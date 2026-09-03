# Runbook

This covers the three things you actually do to this platform: ship a release,
undo one that went wrong, and survive the database failing. Every command here
has been run against a real cluster, and the failover numbers further down are
measured, not estimated.

## Before you start

Set the environment you are working on and confirm you are pointed at it. Most
bad incidents in my experience start with someone running the right command
against the wrong environment.

```bash
export ENV=production
aws eks update-kubeconfig --name fpips-$ENV --region eu-west-1
kubectl config current-context
```

## Deploying

We deploy 5 to 10 times a day during market hours, so the deploy path has to be
boring. Flux reconciles the application chart from the `infrastructure`
repository, which means a deploy is a Git commit and nothing else. Nobody runs
`kubectl apply` against production.

To ship a new image, change the tag in the environment's values file and push.

```bash
# kubernetes/charts/app/values-production.yaml
image:
  tag: "2026.09.03-a1b2c3d"
```

Flux polls the repository every minute. If you do not want to wait, push the
reconciliation yourself.

```bash
flux reconcile source git infrastructure
flux reconcile helmrelease app -n fpips-$ENV-main
```

Watch it land. The HelmRelease goes `Progressing` then `Ready`, and the
deployments roll one tier at a time.

```bash
kubectl get helmrelease app -n fpips-$ENV-main -w
kubectl rollout status deploy/web -n fpips-$ENV-main
```

### What happens to the long running work

The `web` tier drains in 60 seconds and nobody notices. The other two are the
ones to think about.

`cable` holds WebSocket connections, so its pods get a 900 second grace period
and a preStop hook that stops accepting new connections while it finishes the
ones it has. The ALB is configured with a 900 second deregistration delay to
match. A cable rollout is therefore slow on purpose. Do not "fix" it by
shortening the grace period, because that is the same as dropping traders'
connections mid-session.

`worker-critical` runs payout execution and account provisioning, and some of
those jobs run for several minutes. Its preStop hook runs `sidekiqctl quiet`,
which stops the process picking up new jobs while it finishes what it is
holding, and it also carries the `karpenter.sh/do-not-disrupt` annotation so
Karpenter will not evict it during consolidation.

### Schema changes

Migrations do not run as part of the deploy, and this is the single most
important thing in this runbook.

We use expand and contract. A release that needs a schema change ships the
additive migration first, in its own release, with application code that
tolerates both the old and the new shape. The column drop comes in a later
release, once the rollback window has passed. That means at any moment you can
roll the application back one release without the database disagreeing with it.

If you find yourself wanting to run a destructive migration and deploy the code
that depends on it in the same release, stop. That is the change that makes the
two minute rollback below impossible.

## Rolling back

A bad deploy must be fully reversible in under two minutes. There are two ways
to do it and you need to know which one you are on, because Flux owns the
HelmRelease and will undo a manual change at the next reconciliation.

### The normal path

Revert the image tag in Git and force a sync. This keeps Git and the cluster in
agreement and leaves an audit trail of what happened.

```bash
git revert --no-edit HEAD
git push
flux reconcile source git infrastructure
flux reconcile helmrelease app -n fpips-$ENV-main
```

### The emergency path

If reconciliation latency is unacceptable, suspend Flux first, roll back
directly, then repair Git before resuming. The cluster is knowingly out of sync
with Git between the first and last command here, and if you forget the last
step Flux will quietly put the broken release back.

```bash
flux suspend helmrelease app -n fpips-$ENV-main
helm rollback app -n fpips-$ENV-main
# fix the tag in Git and push, then
flux resume helmrelease app -n fpips-$ENV-main
```

Helm rollback is atomic across all six workloads, which is why the chart exists
rather than a pile of manifests. `kubectl rollout undo` would leave you doing
six separate rollbacks and hoping they all worked.

### If the release included a schema change

You still only roll back the application. Expand and contract means the old
code runs against the new schema. Do not reach for Backtrack to undo a deploy.

Backtrack is for the case where a migration actively destroyed data, which is a
different problem with a different answer, covered below.

## Failure scenario: the Aurora writer fails

This is the scenario worked end to end, because the brief puts hard numbers on
it: we may lose at most 60 seconds of data and must be serving again within 5
minutes.

I ran this against a real cluster rather than reasoning about it. The numbers
below are what actually happened.

### What you will see first

The `web` and `worker` pods start failing readiness, and the critical queue
depth climbs because Sidekiq cannot commit. Aurora emits a failover event. The
writer endpoint keeps resolving the whole time, which is the confusing part:
DNS does not disappear, it points at an instance that is not accepting writes
yet.

Confirm it is actually a failover and not something else:

```bash
aws rds describe-events --source-identifier fpips-$ENV-aurora \
  --source-type db-cluster --duration 20 --region eu-west-1
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 \
  --query 'DBClusters[0].DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]' \
  --output text
```

### What you do

For an unplanned failover, nothing. Aurora promotes a replica on its own and
the application reconnects. Your job is to confirm it happened, confirm the
application recovered, and not make it worse by restarting things underneath it.

Resist the urge to roll the deployments. The pods recover on their own once the
new writer accepts connections, and restarting them during a failover just adds
cold connection pools to an already degraded system.

If you need to force a failover, for example to move off a sick instance:

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier fpips-$ENV-aurora \
  --target-db-instance-identifier fpips-$ENV-aurora-reader-0 \
  --region eu-west-1
```

That command is refused with `InvalidDBClusterStateFault` if the cluster is not
`available`. I hit this during testing, immediately after a Terraform apply had
modified the cluster. Check the state first rather than retrying harder:

```bash
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 --query 'DBClusters[0].Status' --output text
```

### What actually happened when I ran it

I ran a probe writing to the writer endpoint every 500 milliseconds, recording
`@@innodb_read_only` on each attempt, and forced a failover from the writer in
`eu-west-1b` to the reader in `eu-west-1a`.

| Moment | Time (UTC) | Elapsed |
| --- | --- | --- |
| Failover issued | 11:38:53.9 | 0s |
| Last successful write on the old writer | 11:39:18.9 | +25.0s |
| First failed write | 11:39:19.5 | +25.6s |
| Writes working again | 11:39:41.8 | +47.9s |
| Last transient failure | 11:39:45.0 | +51.1s |

Three things in there are worth understanding.

**Writes kept succeeding for 25 seconds after I issued the failover.** The API
call returns immediately but the promotion is asynchronous. If you issue a
failover and watch your dashboards, you will see nothing happen for around half
a minute and conclude the command did not work. It did.

**Write unavailability was 22.8 seconds in one block**, then two brief flaps of
0.5 and 1.6 seconds as connections settled on the new writer. Total write
downtime 24.8 seconds, fully stable at 51.6 seconds after the trigger.

**Not a single acknowledged write was lost.** The largest gap between
consecutive rows persisted in Aurora was 23.348 seconds, which is the outage
itself, and there are no gaps anywhere else. Every write the database
acknowledged before the failover was there afterwards.

### Against the requirement

| Requirement | Measured | Margin |
| --- | --- | --- |
| RTO under 5 minutes | 51.6 seconds worst case | 5.8x |
| RPO under 60 seconds | 0 seconds of data lost | met outright |

The RPO result is not luck and it is worth saying why, because it changes how
you think about the rest of the design. Aurora does not replicate between
instances the way classic MySQL does. The storage layer is shared and written
six ways across three availability zones, and instances are compute attached to
it. Promoting a replica does not replay a log, it points a different compute
node at storage that already has every committed transaction. There is no
window in which an acknowledged write exists on one instance and not another.

### What this does not protect you from

Multi-AZ failover handles an instance dying. It does nothing for logical
damage, because a migration that drops the wrong column is faithfully applied
to all six copies of the storage instantly.

That is what the 24 hour Backtrack window is for. It rewinds the cluster in
place to a point in time in minutes, without restoring a snapshot.

```bash
# check the window you actually have
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 --query 'DBClusters[0].[EarliestBacktrackTime,BacktrackWindow]'

# rewind, after taking the application down
aws rds backtrack-db-cluster --db-cluster-identifier fpips-$ENV-aurora \
  --backtrack-to 2026-09-03T11:30:00Z --region eu-west-1
```

Backtrack rewinds the entire cluster, so every write after that timestamp is
gone, not just the damaging one. Scale the application to zero first, decide
the timestamp deliberately, and treat it as a last resort rather than an undo
button.

### Verifying afterwards

```bash
# the promoted instance is now the writer, in the other AZ
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 \
  --query 'DBClusters[0].DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]' \
  --output text

# the critical queue drained rather than merely stopped growing
kubectl get hpa keda-hpa-worker-critical -n fpips-$ENV-main

# nothing is stuck
kubectl get pods -n fpips-$ENV-main --field-selector=status.phase!=Running
```

Aurora does not fail back on its own. After an unplanned failover your writer
is in a different AZ than it started, which is fine and needs no action. The
promotion tiers decide who gets promoted next time: customer facing readers sit
at tier 1, and the finance reporting replica sits at tier 15 precisely so that
a replica busy running a month end report is never what customers land on.

## Tearing an environment down

Destroy in reverse order: `addons`, then `platform`. Never the other way, and
never `bootstrap`, which holds the state bucket and the shared deploy key.

```bash
make destroy ENV=dev LAYER=addons
make destroy ENV=dev LAYER=platform
```

Remove the application workloads first, so pods drain and Karpenter releases
nodes while its controller is still running.

```bash
kubectl delete helmrelease app -n fpips-$ENV-main
```

**Expect the addons destroy to need two passes on a busy cluster.** Karpenter
puts a finalizer on its NodePools and holds it until every node it created is
drained and terminated. On a cluster with several nodes that can outrun the
Helm uninstall timeout, and Terraform reports:

```
Error: Error uninstalling release
```

That is not a broken teardown. Karpenter finishes the drain in its own time.
Re-run the destroy and it completes in seconds. Do not start deleting EC2
instances by hand: that strands a NodeClaim holding a finalizer nothing is left
to clear, which is a genuinely awkward state to get out of.

Confirm nothing was orphaned before you walk away. Karpenter provisioned nodes
are not Terraform managed, so this is the check that matters:

```bash
aws ec2 describe-instances --region eu-west-1   --filters "Name=tag-key,Values=karpenter.sh/nodepool"              "Name=instance-state-name,Values=running,pending"   --query 'Reservations[].Instances[].InstanceId' --output text
```

Two things survive a destroy and keep costing money. KMS keys are scheduled for
deletion rather than deleted, with a 30 day window, and continue to bill for
that period. Elastic IPs in staging and production carry `prevent_destroy`,
because releasing an address a payment provider has allowlisted is a five
business day mistake, so Terraform will refuse to remove them by design.
