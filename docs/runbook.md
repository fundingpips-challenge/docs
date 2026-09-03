# Runbook

Three paths, and using the wrong one is the most common mistake:

| Change | Path |
| --- | --- |
| Application image or config | Git commit, Flux reconciles |
| Infrastructure | GitHub Actions Terraform pipeline |
| Incident response | Manual, by a human, commands below |

Nobody runs `terraform apply` from a laptop. The CI role is scoped to the
repository through OIDC and staging and production sit behind environment
protection, so a local apply bypasses the review that exists for a reason.

## Deploying

### Application

Change the image tag and merge.

```yaml
# kubernetes/charts/app/values-production.yaml
image:
  tag: "2026.09.03-a1b2c3d"
```

Flux polls every minute and reconciles. Watch it land:

```bash
kubectl get helmrelease app -n fpips-$ENV-main -w
```

To skip the poll interval:

```bash
flux reconcile source git infrastructure
flux reconcile helmrelease app -n fpips-$ENV-main
```

### Infrastructure

Open a PR. `checks.yml` runs Trivy, `fmt`, `validate` across all four layers,
and a plan per environment commented on the PR. Trivy fails the build on
CRITICAL or HIGH and is not allowed to fail. The same workflow also runs on
demand, against one environment or all three, without a PR:

```
Actions -> Terraform checks -> Run workflow -> choose environment
```

Applying is always manual, every environment, including dev. Merging does not
apply anything by itself.

```
Actions -> Terraform -> Run workflow -> action: apply -> choose environment
```

`terraform.yml` takes `action` (`apply` or `destroy`) as an input rather than
being two separate workflows, so there is one place to look and one history to
read. Layers run as chained jobs, so `platform` completes before `addons`
starts and `app` last. That ordering is enforced by the pipeline, not by
remembering it.

### What deploys slowly on purpose

`web` drains in 60 seconds. The other two are deliberate:

- **`cable`** holds WebSockets. 900 second grace period, a preStop hook that
  stops accepting new connections, and a matching 900 second ALB deregistration
  delay. Shortening this drops traders mid-session.
- **`worker-critical`** runs payouts for several minutes. preStop runs
  `sidekiqctl quiet`, and `karpenter.sh/do-not-disrupt` stops consolidation
  evicting it.

### Schema changes

Migrations do not run in the deploy. Expand and contract: additive migration in
its own release with code tolerating both shapes, drop the column later once the
rollback window has passed.

If you want a destructive migration and the code depending on it in one release,
stop. That is the change that makes the rollback below impossible.

## Rolling back

### Normal

Revert the commit and let the pipeline do it.

```bash
git revert --no-edit <sha>
git push
```

For an application change Flux picks it up within a minute. For infrastructure,
revert the commit, then run `terraform.yml` with `action: apply` against the
affected environment; nothing applies on its own.

### Break glass

Only when the pipeline itself is unavailable: GitHub down, workflow broken,
credentials expired. This bypasses review, so say so in the incident channel.

Flux owns the HelmRelease and will revert a manual change at the next
reconciliation, so suspend first and repair Git before resuming.

```bash
flux suspend helmrelease app -n fpips-$ENV-main
helm rollback app -n fpips-$ENV-main
# fix the tag in Git and push, then
flux resume helmrelease app -n fpips-$ENV-main
```

`helm rollback` is atomic across all six workloads. `kubectl rollout undo` would
be six separate rollbacks and a hope, which is why the chart exists.

If the release included a schema change you still roll back only the
application. Expand and contract means old code runs against the new schema. Do
not reach for Backtrack to undo a deploy.

## Failure scenario: the Aurora writer fails

An unplanned failure is incident response: manual, because nobody chooses
when it happens. Deliberately testing failover is a pipeline, `failover-test.yml`,
manual trigger only and never on a PR or a push, since it forces a real
failover on a real cluster:

```
Actions -> Aurora failover test -> Run workflow -> choose environment
```

It runs the same probe as below, forces the failover, and fails the job if
downtime exceeds the 5 minute RTO. Needs a reader instance to fail over to;
set `db_reader_count > 0` for that environment first if it does not have one.

### What you see

`web` and `worker` pods fail readiness and the critical queue climbs. The writer
endpoint keeps resolving throughout, which is the confusing part: DNS does not
disappear, it points at an instance not yet accepting writes.

```bash
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 \
  --query 'DBClusters[0].DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]' \
  --output text
```

### What you do

For an unplanned failover, nothing. Aurora promotes a replica and the
application reconnects. Confirm it happened and do not restart pods underneath
it; they recover on their own, and restarting adds cold connection pools to a
degraded system.

To force one outside the pipeline, e.g. the pipeline itself is unavailable:

```bash
aws rds failover-db-cluster \
  --db-cluster-identifier fpips-$ENV-aurora \
  --target-db-instance-identifier fpips-$ENV-aurora-reader-0 \
  --region eu-west-1
```

This is refused with `InvalidDBClusterStateFault` if the cluster is not
`available`, which happens right after any change. Check first rather than
retrying:

```bash
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 --query 'DBClusters[0].Status' --output text
```

### Measured, not estimated

A probe writing every 500ms while forcing a failover from `eu-west-1b` to
`eu-west-1a`:

| Moment | Elapsed |
| --- | --- |
| Failover issued | 0s |
| Last write on the old writer | +25.0s |
| First failed write | +25.6s |
| Writes working again | +47.9s |
| Last transient failure | +51.1s |

Write unavailability 24.8 seconds, stable at 51.6 seconds, **zero acknowledged
writes lost**. Largest gap between persisted rows was 23.3 seconds, the outage
itself, with no gaps elsewhere.

Against the requirement: RTO 5 minutes met with 5.8x margin, RPO 60 seconds met
outright at zero.

**Writes kept succeeding for 25 seconds after the command.** The API returns
immediately and promotion is asynchronous. Watching a dashboard you would
conclude it failed and run it again.

### What this does not cover

Multi-AZ handles an instance dying, not logical damage. A migration dropping the
wrong column reaches all six storage copies instantly. That is what the 24 hour
Backtrack window is for.

```bash
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 --query 'DBClusters[0].[EarliestBacktrackTime,BacktrackWindow]'

aws rds backtrack-db-cluster --db-cluster-identifier fpips-$ENV-aurora \
  --backtrack-to 2026-09-03T11:30:00Z --region eu-west-1
```

Backtrack rewinds the whole cluster, so every write after that timestamp is
gone. Scale the application to zero first and treat it as a last resort.

### Verifying

```bash
aws rds describe-db-clusters --db-cluster-identifier fpips-$ENV-aurora \
  --region eu-west-1 \
  --query 'DBClusters[0].DBClusterMembers[].[DBInstanceIdentifier,IsClusterWriter]' \
  --output text

kubectl get hpa keda-hpa-worker-critical -n fpips-$ENV-main
kubectl get pods -n fpips-$ENV-main --field-selector=status.phase!=Running
```

Aurora does not fail back. Your writer is now in a different AZ, which needs no
action. Promotion tiers decide who is next: customer facing readers at tier 1,
the finance reporting replica at tier 15 so a replica running a month end report
is never what customers land on.

## Tearing an environment down

```
Actions -> Terraform -> Run workflow -> action: destroy -> choose environment
```

Dev and staging only, and you must retype the environment name to confirm.
Production is blocked in the pipeline's guard job even though it is a valid
choice in the dropdown; removing it is a decision that should not be one click
away.

The workflow removes application workloads first so Karpenter releases its nodes
while its controller still runs, destroys `addons` then `platform`, and asserts
no Karpenter instances survive. Karpenter nodes are not Terraform managed, so
that last check is the one that matters.

The addons destroy retries once by design. Karpenter holds a finalizer on its
NodePools until every node it created is drained, which can outrun the Helm
uninstall timeout and produce `Error: Error uninstalling release`. That is not a
broken teardown. Never delete the EC2 instances by hand: that strands a
NodeClaim holding a finalizer with nothing left to clear it.

If `destroy platform` fails on `DependencyViolation` deleting a subnet or
security group, look for an orphaned ENI first:

```bash
aws ec2 describe-network-interfaces --region eu-west-1 \
  --filters "Name=vpc-id,Values=<vpc-id>" \
  --query 'NetworkInterfaces[].[NetworkInterfaceId,Status,Description]'
```

The VPC CNI occasionally leaves one behind when a node terminates abruptly.
One with `Status: available` (detached) and no live instance behind its
description is safe to delete by hand; re-running the destroy then clears the
subnet and security group that were waiting on it.

Two things survive a destroy and keep costing money. KMS keys are scheduled for
deletion with a 30 day window. Elastic IPs in staging and production carry
`prevent_destroy`, because releasing an address a payment provider has
allowlisted is a five business day mistake.
