# Design

This describes the infrastructure for a proprietary trading platform on AWS:
three processes, three environments, and the reasoning behind the choices that
were not obvious. Where I tested something rather than assumed it, I have said
so and given the numbers.

## What I assumed, and what I would have asked

Several things were deliberately unspecified. These are the assumptions I made
and would confirm before building this for real.

**Kubernetes on EKS.** The brief never says containers. Three long lived
processes with different scaling signals and a two minute rollback requirement
is an orchestrator shaped problem, and EKS is the boring answer. ECS would work
and be cheaper to operate; I would ask whether the team already runs Kubernetes
before committing.

**One AWS account for all three environments,** with VPC level isolation. For
a firm handling payouts I would want an account per environment in an
Organization, so a mistake in dev cannot reach production credentials. Single
account keeps this exercise reviewable; I would raise the change on day one.

**Single region.** No region loss RTO was given, so I did not build for one.

The five questions I would have sent before writing any Terraform:

1. Single account, or an Organization with an account per environment?
2. Is there a DR tier or RTO for full region loss? This decides whether Aurora
   Global Database is in scope.
3. Who owns application deploy today? This bounds where Terraform stops and is
   the largest scoping question in the brief.
4. Does the 4 TB carry PII? This decides whether staging can clone production.
5. Are the 4x and 3x traffic multipliers on requests per second, on concurrent
   connections, or both? `web` and `cable` scale on different signals and the
   answer changes both.

## The shape of it

Four Terraform layers, applied in order, each with its own state:

```
bootstrap  ->  platform  ->  addons  ->  app
```

`bootstrap` runs once per account and creates the state bucket and the Git
deploy key. `platform` is the per environment baseline: VPC, EKS, Aurora,
Valkey, Secrets Manager. `addons` installs the in cluster platform components
through Helm. `app` holds per instance AWS resources.

Three layers rather than two is a consequence of provider bootstrapping, not
taste. Terraform configures providers before it plans, so the Helm and
Kubernetes providers cannot be configured from a cluster endpoint that does not
exist yet. Cluster lifecycle and in cluster resources have to be separate
applies.

Compute is split deliberately. Karpenter, CoreDNS and metrics-server run on
Fargate; everything else runs on EC2 that Karpenter provisions. The reason is
circularity: the thing that provisions nodes must not need a node to be healthy.
Fargate hosts only that small always on set, selected by pod label rather than
namespace, so the rest of `kube-system` still lands on cheaper, denser EC2.

Five Karpenter NodePools: one untainted `system` pool and four tainted
application pools, on Bottlerocket and arm64.

## Decisions

### Traffic: a 4x daily swing and 3x spikes in under two minutes

Two scaling signals, because the load has two characters. KEDA drives both.

A **cron trigger** holds a floor of 12 web replicas through market hours. The
daily peak is predictable to within 30 minutes, so front loading it means the
capacity is already there when the ramp starts, instead of reactive scaling
chasing a curve it always trails.

A **CPU trigger** handles the unpredictable part. Scale up allows 100 percent
growth every 15 seconds with no stabilisation window, so 3 replicas become 48
inside a minute. Scale down is deliberately asymmetric at 10 percent per minute
behind a five minute window: scaling down fast during what turns out to be the
gap between two news events is how you cause the outage you were avoiding.

Rejected: HPA alone. It cannot express "be big at 07:00 because the market
opens", so every morning would be a reactive catch up.

### Queues: critical under 60 seconds at p95

Four separate Deployments, one per queue, not one worker reading four queues.
Sidekiq weights queues within a process, but a process is a shared thread pool:
a flood of `competition` jobs consumes threads `critical` then waits for.
Separate deployments make that impossible.

The numbers differ because the requirements differ:

| Queue | Node pool | min / max | Scale at | Poll | Grace |
| --- | --- | --- | --- | --- | --- |
| critical | on-demand | 4 / 40 | 5 jobs | 5s | 900s |
| default | spot | 2 / 30 | 50 jobs | 15s | 600s |
| low | spot | 0 / 15 | 200 jobs | 30s | 600s |
| competition | spot | 0 / 20 | 200 jobs | 30s | 600s |

`critical` never scales to zero, polls three times more often, and triggers at 5
queued jobs rather than 50. That is the p95 requirement expressed as
configuration rather than as a hope.

It also runs on demand only and carries `karpenter.sh/do-not-disrupt`, because
payout jobs run for several minutes and consolidation would otherwise evict one
mid execution. A Terraform validation block rejects any attempt to put
`critical` or `cable` on spot, with the reason in the error message.

**Never lost, never executed twice** is three mechanisms, not one. Sidekiq
Enterprise unique jobs prevents double enqueue. Idempotent handlers make a retry
safe when it happens anyway. Sidekiq Pro `super_fetch` returns a job to the
queue if the worker dies holding it, which is the case plain Sidekiq loses.

### Data: RPO 60 seconds, RTO 5 minutes

Aurora MySQL, Multi-AZ, single region. **I tested this rather than asserting
it.** Forcing a failover on a live cluster while writing every 500ms:

| Measure | Result |
| --- | --- |
| Write unavailability | 24.8 seconds |
| Fully stable after | 51.6 seconds |
| Acknowledged writes lost | 0 |

RTO is met with 5.8x margin. RPO is not merely met, it is zero, and the reason
matters: Aurora does not replicate between instances. Storage is shared and
written six ways across three availability zones, and instances are compute
attached to it. Promotion points a different compute node at storage that
already holds every committed transaction.

One detail found by hitting it: writes kept succeeding for 25 seconds after
the failover was issued, because the API returns immediately and promotion is
asynchronous. An operator watching dashboards would conclude it failed and run
it again.

Rejected: **Aurora Global Database.** It buys a region loss tier the brief does
not ask for, at the cost of a second cluster and a managed failover process with
its own RTO. It is the upgrade path if a region loss RTO is ever named.

**Finance reporting isolation is physical, not conventional.** The built in
reader endpoint load balances across every replica including the reporting one,
which is exactly what the brief forbids, so the application never uses it. There
are two custom endpoints instead: one containing every reader except the
reporting replica, and one containing only it. The reporting replica also sits
at `promotion_tier = 15`, so an instance busy with a month end report is never
what customers fail over onto, and it has its own parameter group bounding
`max_connections` and `max_execution_time`. A runaway BI query can exhaust its
own replica and nothing else.

**Aurora Backtrack, 24 hours.** Multi-AZ does nothing for logical damage: a
migration that drops the wrong column is applied faithfully to all six copies
instantly. Backtrack rewinds the cluster in place in minutes.

**I/O-Optimized storage.** At 4 TB serving thousands of reads per second, flat
rate I/O is cheaper and predictable. Below roughly 25 percent of spend going to
I/O, standard is cheaper, which is why dev uses standard.

### Caching: reference data read thousands of times a second

Two Valkey replication groups, not one, and this is the decision I would defend
hardest.

Sidekiq requires `maxmemory-policy noeviction`, because evicting a queue key
loses the job silently. Reference data caching wants `allkeys-lru` to shed cold
keys under pressure. Those cannot coexist on one group, and running the queue on
an LRU group is a direct route to violating "a job must never be silently lost".
So: a `cache` group with `allkeys-lru` and no snapshots, and a `queue` group
with `noeviction` holding Sidekiq and sessions, snapshotted daily. A validation
block rejects any queue group configured to evict.

**AOF was investigated and rejected, because it is not available.** AWS
documents `appendonly` as unsupported on Redis OSS 2.8.22 and later, and that
ElastiCache Multi-AZ and AOF are mutually exclusive. Enabling it would have cost
the failover that actually satisfies "must survive the loss of a single node".
Durability is three layers instead: Multi-AZ failover, daily snapshots, and
Sidekiq `super_fetch` for a worker dying mid job.

### Network: static egress, and a database an auditor can check in a minute

Three subnet tiers. The database tier's route table contains the VPC local
route and an S3 gateway endpoint, and nothing else. There is no `aws_route`
resource pointing at it anywhere in the repository.

That is the auditor answer, and it is three commands on a live cluster:

```
route table       ->  10.30.0.0/16 local, vpce-... (S3 gateway)
PubliclyAccessible ->  False
ingress on 3306   ->  one rule, from the node security group, zero CIDR rules
```

The third is the strongest. There is no address range permitted to reach the
database, only a named security group. You cannot misconfigure your way to
internet exposure from there without deliberately adding a route.

**Egress addresses are pinned.** One NAT per AZ means a set of allowlisted
addresses, and Terraform allocates spares beyond those in use with
`prevent_destroy` on the whole set. Adding an AZ or replacing a NAT never starts
a new five day notice cycle, because the address was allowlisted up front and
cannot be released. Dev is exempt: it talks to sandboxes and is rebuilt often.

### Secrets: 14 at boot, 2 rotated quarterly

External Secrets Operator syncs from Secrets Manager. All 14 are named
explicitly in the module rather than counted: Aurora master and application
credentials, both Valkey auth tokens, the Sidekiq licence, four Rails keys,
two payment provider keys, KYC, market data, and the OTel ingest token.

The two rotating quarterly are the payment provider keys, which is a neat
coincidence: they are the same credentials tied to the egress IP allowlist, so
both third party constraints land on one relationship.

Terraform writes only the three values it generates. Third party keys get an
encrypted container and a rotation tag, and a human populates the value.
Terraform should not be where a payment provider key gets typed.

**Rotation uses Stakater Reloader**, restarting pods when the synced Secret
changes. Rejected: mounting secrets as files and having the app watch them. That
avoids a restart but needs application changes the platform team does not own,
and adds a rarely exercised reload path. We roll 5 to 10 times a day already, so
a rolling restart is well exercised, and this is 8 restarts a year.

### Identity: two mechanisms, deliberately

EKS Pod Identity for everything on EC2. IRSA for Karpenter alone, because
Karpenter runs on Fargate and AWS documents that Pod Identity is unavailable to
Fargate pods. Writing this up as an exception with a reason is better than
pretending the cluster is uniform.

Rejected: moving the cluster critical add-ons to a small managed node group so
Pod Identity could be used everywhere. It also avoids the circular dependency
and would be simpler, at the cost of two always on instances per environment.

### Deployment: 5 to 10 a day, reversible in under two minutes

Flux reconciles a Helm chart from Git. A deploy is a commit; nobody runs
`kubectl apply` at production.

The chart matters for the rollback requirement specifically. `helm rollback` is
one atomic operation across all six workloads with a named revision.
`kubectl rollout undo` would be six separate rollbacks and a hope.

Because Flux owns the release, a manual rollback is reverted at the next
reconciliation. The runbook documents two paths and says which is which: revert
in Git normally, or suspend, roll back, repair Git for emergencies.

**Schema changes use expand and contract.** Additive migration first, in its own
release, with code tolerating both shapes; the column drop comes later once the
rollback window has passed. This is the actual answer to "reversible in under
two minutes even with schema changes". Running migrations in an init container
does not solve it, because it makes the rollback path depend on reversing a
migration under time pressure.

### Observability: "the site is slow", answered in minutes

OpenTelemetry, with RED metrics per tier and a trace spanning request, queue
wait and database call. The value is not that OTel is installed, it is that the
question decomposes: is the ALB seeing latency, is `web` slow, is `web` waiting
on the database, or is work sitting in a queue. Each is a different hop with its
own owner and fix. Queue latency is also what KEDA scales on, so the p95 SLO and
the autoscaler read the same number.

## What it costs

Prices below are on-demand for eu-west-1, pulled from the AWS Pricing API in
September 2026, at 730 hours a month.

| Component | Detail | Monthly |
| --- | --- | --- |
| Aurora instances | 3x db.r6g.4xlarge at $2.291/hr, 1x db.r6g.2xlarge at $1.146/hr | $5,854 |
| Aurora storage | 4 TB I/O-Optimized at $0.248/GB-mo | $1,016 |
| Valkey | 6x cache.r7g.large at $0.1944/hr | $851 |
| NAT gateways | 3 at $0.048/hr, plus $0.048/GB processed | $105 |
| EKS control plane | $0.10/hr | $73 |
| KMS and Secrets Manager | 6 keys, 14 secrets | $12 |
| **Fixed subtotal** | | **~$7,900** |

Karpenter compute is on top and is the part that varies. It is also where the
"must not cost peak money at 3am" requirement is actually answered.

**The 3am argument, quantified.** Take the web tier at a 4x swing: 12 replicas
at peak, 3 overnight, which is roughly 3 m7g.xlarge nodes falling to 1.

- Flat at peak: 3 nodes x 730 hr x $0.1819 = **$398/month**
- Scheduled floor plus consolidation: 242 market hours at 3 nodes, 488 hours at
  1 node = 1,214 node-hours = **$221/month**

That is a 44 percent saving on the web tier from the scaling profile alone,
before spot. The `default`, `low` and `competition` worker pools run spot
preferred, which is typically another 70 percent off their compute, and `low`
and `competition` scale to zero entirely when idle.

**Why I/O-Optimized.** Standard storage is $0.110/GB-mo, so 4 TB is $451, an
apparent $565 saving. But standard also bills I/O at $0.20 per million requests.
At a sustained 5,000 reads per second that is about $2,592 a month. The
crossover is near 1,100 requests per second; "thousands per second" is well past
it. Dev uses standard because dev lacks that traffic.

Aurora is roughly 87 percent of fixed cost. If this bill needed reducing,
reserved instances on the three customer facing nodes take about 34 percent off
them at one year no upfront, which beats any amount of Kubernetes tuning.

## How it fails

I built this against a real AWS account, and the most useful thing that produced
was a list of failures that `fmt`, `validate` and `plan` all passed cleanly. Two
left the cluster fully green and fundamentally broken.

**No security group rules between Karpenter nodes and the EKS managed cluster
security group.** Fargate pods carry the cluster SG, Karpenter nodes carry only
the node SG, and nothing connected them. CoreDNS runs on Fargate, so every EC2
pod had no DNS: the application could not have resolved Aurora or Valkey, and
metrics-server could not scrape kubelet, silently disabling CPU scaling. All 125
resources applied successfully.

**A webhook that bricked the cluster for its own remedy.** The AWS Load Balancer
Controller registers a webhook intercepting every Service creation, with
`failurePolicy: Fail` and no selector. Its pods were pending with no EC2 node,
so the webhook had no endpoints, so every Service creation failed. That took
down Karpenter, which creates a Service, which is what would have provisioned
the node the controller needed. Disabled it, since we use Ingress and never
Service type LoadBalancer.

**IRSA config against a Pod Identity trust policy.** The ClusterSecretStore
specified `auth.jwt`, making ESO call `AssumeRoleWithWebIdentity` while the role
trusted `pods.eks.amazonaws.com`. The same class of mistake as leaving an IRSA
annotation on a service account after migrating.

**Stale pins, three times.** Five Helm charts out of date, one by two major
versions. An Aurora engine version that no longer existed in the region. A
deprecated provider attribute. Pinning is correct for a database, but pins rot
silently and nothing short of apply notices.

| Failure | Behaviour | Recovery |
| --- | --- | --- |
| Aurora writer dies | 25s of write errors, promoted at ~50s | automatic, RPO 0 measured |
| AZ lost | one NAT and its nodes go, Karpenter reprovisions | automatic, egress IPs pre-allowlisted |
| Valkey primary dies | Multi-AZ failover, super_fetch returns in flight jobs | automatic |
| Spot reclaim | 2 minute warning to the interruption queue, drained | automatic, critical never on spot |
| No spot capacity | pods Pending, existing capacity keeps serving | widen instance families |
| Bad deploy | rollback under 2 minutes | Helm rollback or Git revert |
| Bad migration | expand and contract, old code runs | Backtrack if data destroyed |
| Flux broken | cluster keeps last reconciled state | fix Flux, nothing customer facing |

The one that worries me most is organisational rather than technical. Expand and
contract only works if every engineer follows it, and the first destructive
migration shipped alongside the code depending on it is the day the two minute
rollback stops being true. That deserves a CI check rejecting destructive DDL,
which I have not built.

## At real scale, I would do this differently

**State per component, not per layer.** Splitting network, EKS, RDS, ElastiCache
and application infrastructure into separate states gives real blast radius
separation and independent apply cadence. Four layers here costs a documented
apply order and a pile of remote state reads; for a submission read in an hour,
that spends the reviewer's patience on plumbing rather than reasoning.

**Account per environment in an Organization,** with state and the deploy key in
a tooling account. The single account compromise is the first thing I would
change.

**Separate repositories** for modules, environment config and cluster state,
with a promotion pipeline moving a tested module version dev to staging to
production. That is the pattern for a platform team serving several product
teams, and more machinery than one exercise justifies.

**A private Kubernetes API endpoint** with self hosted runners in the VPC. I
kept it public with private access on and IAM access entries, because network
position is the right control for a database and defence in depth for an IAM
authenticated API. Tunnelling through a bastion does not work cleanly: the Helm
provider then faces a certificate whose SAN is the real EKS DNS name.
