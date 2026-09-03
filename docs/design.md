# Design

Infrastructure for a trading platform on AWS: three processes, three
environments. Measured results are marked as such.

## Assumptions

| Assumption | Why | Would confirm |
| --- | --- | --- |
| EKS | Three processes, different scaling signals, 2 minute rollback | Does the team already run Kubernetes? ECS is cheaper to operate |
| One account, VPC isolation | Keeps the exercise reviewable | Account per environment is correct for payouts |
| Single region | No region loss RTO was given | If one exists, Aurora Global Database returns |

Questions I would have sent first:

1. One account or an Organization?
2. Is there a region loss RTO?
3. Who owns application deploy? This bounds where Terraform stops.
4. Does the 4 TB carry PII? Decides if staging can clone production.
5. Are the 4x and 3x multipliers requests, connections, or both? `web` and
   `cable` scale on different signals.

## Structure

```
bootstrap -> platform -> addons -> app
```

Each layer has its own state. `bootstrap` runs once per account: state bucket,
Git deploy key, CI role. `platform` is the per environment baseline. `addons`
installs in cluster components via Helm. `app` holds per instance resources.

Three layers rather than two is forced: Terraform configures providers before it
plans, so Helm and Kubernetes providers cannot point at a cluster that does not
exist yet.

One `environments/<env>.tfvars` per environment, read by every layer. Layers
warn about keys they do not declare, which is expected.

Karpenter, CoreDNS and metrics-server run on Fargate; everything else on EC2
that Karpenter provisions. The thing that provisions nodes must not need a node.
Fargate selection is by pod label, not namespace, so the rest of `kube-system`
stays on cheaper EC2. Five NodePools on Bottlerocket arm64: one untainted
`system`, four tainted application pools.

## Decisions

### Traffic

Two KEDA triggers, because the load has two characters.

- **Cron** holds 12 web replicas through market hours. The peak is predictable
  to 30 minutes, so front load it rather than let reactive scaling trail it.
- **CPU** handles spikes. 100 percent growth per 15 seconds, no stabilisation
  window, so 3 replicas reach 48 in a minute. Scale down is asymmetric at 10
  percent per minute behind a 5 minute window, because scaling down into what
  turns out to be a gap between news events causes the outage you were avoiding.

Rejected: HPA alone. It cannot express "be big at 07:00".

### Queues

Four Deployments, one per queue. Sidekiq weights queues inside a process, but a
process is one thread pool: a `competition` flood consumes threads `critical`
waits for.

| Queue | Pool | min/max | Scale at | Poll | Grace |
| --- | --- | --- | --- | --- | --- |
| critical | on-demand | 4 / 40 | 5 jobs | 5s | 900s |
| default | spot | 2 / 30 | 50 jobs | 15s | 600s |
| low | spot | 0 / 15 | 200 jobs | 30s | 600s |
| competition | spot | 0 / 20 | 200 jobs | 30s | 600s |

`critical` never scales to zero, polls 3x more often, triggers at 5 jobs not 50,
runs on demand only, and carries `karpenter.sh/do-not-disrupt` so consolidation
cannot evict a running payout. A validation block rejects `critical` or `cable`
on spot.

Never lost, never twice, is three mechanisms: Sidekiq Enterprise unique jobs
prevents double enqueue, idempotent handlers make retries safe, Sidekiq Pro
`super_fetch` returns jobs held by a worker that died.

### Data

Aurora MySQL, Multi-AZ, single region. **Tested, not assumed.** Forced failover
while writing every 500ms:

| Measure | Result | Requirement |
| --- | --- | --- |
| Write unavailability | 24.8s | |
| Fully stable | 51.6s | RTO 5 min, 5.8x margin |
| Writes lost | 0 | RPO 60s, met outright |

RPO is zero because Aurora does not replicate between instances. Storage is
shared, written six ways across three AZs; promotion points new compute at
storage that already holds every commit.

Found by testing: writes kept succeeding for 25s after the failover was issued,
because promotion is asynchronous. An operator would think the command failed.

**Finance isolation is physical.** The built in reader endpoint includes the
reporting replica, which the brief forbids, so the application never uses it.
Two custom endpoints instead: one excluding the reporting replica, one
containing only it. The reporting replica sits at `promotion_tier = 15` so it is
never promoted, and has its own parameter group capping `max_connections` and
`max_execution_time`.

**Backtrack, 24 hours.** Multi-AZ does nothing for logical damage; a bad
migration reaches all six copies instantly.

**I/O-Optimized.** Standard is $0.110/GB-mo plus $0.20 per million I/O. At 5,000
reads/sec that is roughly $2,592/month against a $565 storage saving. Crossover
is near 1,100 reads/sec. Dev uses standard.

Rejected: Aurora Global Database. Buys a region loss tier nobody asked for.

### Caching

Two Valkey groups, not one. Sidekiq needs `noeviction` or it loses jobs
silently. Reference caching needs `allkeys-lru` to shed cold keys. These cannot
coexist, and an LRU queue violates "never silently lost". So a `cache` group
(LRU, no snapshots) and a `queue` group (noeviction, snapshotted). A validation
block rejects an evicting queue group.

**AOF was rejected because it does not exist.** AWS documents `appendonly` as
unsupported past Redis 2.8.22, and Multi-AZ and AOF as mutually exclusive.
Enabling it would have cost the failover that satisfies the actual requirement.
Durability is Multi-AZ failover, daily snapshots, and `super_fetch`.

### Network

Three subnet tiers. The database tier's route table holds the VPC local route
and an S3 gateway endpoint. No `aws_route` points at it anywhere in the repo.

The auditor answer is three commands:

```
route table         -> local + S3 gateway endpoint only
PubliclyAccessible  -> False
ingress on 3306     -> one security group, zero CIDR rules
```

The third is strongest: no address range can reach the database, only a named
security group.

**Egress IPs are pinned.** One NAT per AZ, all addresses allowlisted, plus
spares with `prevent_destroy`. Adding an AZ never triggers a new five day notice
cycle. Dev is exempt.

### Secrets

ESO syncs 14 named secrets from Secrets Manager. Terraform writes only the three
it generates; third party keys get an encrypted container and a human fills the
value. The two rotating quarterly are the payment provider keys, the same ones
tied to the egress allowlist.

**Reloader** restarts pods on secret change. Rejected: file mounts with app side
watching, because it needs application changes the platform team does not own
and adds a rarely exercised path. We roll 5 to 10 times daily anyway; this is 8
restarts a year.

### Identity

Pod Identity everywhere on EC2. IRSA for Karpenter alone, because it runs on
Fargate and AWS documents Pod Identity as unavailable there. An exception with a
reason beats pretending the cluster is uniform.

Rejected: a small managed node group for cluster critical add-ons, allowing
uniform Pod Identity at the cost of two always on instances per environment.

### Deployment

Flux reconciles a Helm chart from Git. A deploy is a commit.

The chart exists for the rollback requirement: `helm rollback` is one atomic
operation across six workloads. `kubectl rollout undo` would be six rollbacks
and a hope.

Flux owns the release, so a manual rollback is reverted at the next
reconciliation. The runbook documents both paths and which to use.

**Expand and contract** for schema changes: additive migration first in its own
release, drop later once the rollback window passes. This is the real answer to
"reversible in two minutes with schema changes".

### Observability

OpenTelemetry, RED metrics per tier, one trace spanning request, queue wait and
database call. The value is that "the site is slow" decomposes into a specific
hop with an owner. Queue latency is also what KEDA scales on, so the SLO and the
autoscaler read the same number.

## Cost

On-demand, eu-west-1, from the AWS Pricing API, 730 hours.

| Component | Detail | Monthly |
| --- | --- | --- |
| Aurora instances | 3x db.r6g.4xlarge, 1x db.r6g.2xlarge | $5,854 |
| Aurora storage | 4 TB I/O-Optimized at $0.248/GB | $1,016 |
| Valkey | 6x cache.r7g.large at $0.1944/hr | $851 |
| NAT gateways | 3 at $0.048/hr | $105 |
| EKS control plane | $0.10/hr | $73 |
| KMS, Secrets Manager | | $12 |
| **Fixed** | | **~$7,900** |

**Not paying peak at 3am**, quantified. Web tier at a 4x swing, 12 replicas to
3, roughly 3 m7g.xlarge to 1:

- Flat at peak: 3 x 730 x $0.1819 = **$398/month**
- Scheduled floor: 242 market hours at 3 nodes, 488 at 1 = **$221/month**

44 percent off the web tier before spot. `default`, `low` and `competition` run
spot preferred, another ~70 percent off their compute, and the last two scale to
zero.

Aurora is 87 percent of fixed cost. Reserved instances on the three customer
facing nodes take ~34 percent off them, which beats any Kubernetes tuning.

## How it fails

Built against a real account. Four defects passed `fmt`, `validate` and `plan`
cleanly; two left a green apply over a broken cluster.

| Defect | Effect |
| --- | --- |
| No SG rules between Karpenter nodes and the EKS managed cluster SG | No DNS for any EC2 pod. The app could not resolve Aurora or Valkey. metrics-server could not scrape kubelet, silently disabling CPU scaling |
| ALB controller webhook on all Services, `failurePolicy: Fail`, no selector | Its own pods were pending, so the webhook had no endpoints, so every Service creation failed, including Karpenter's, which would have provided the node it needed |
| ClusterSecretStore using `auth.jwt` against a Pod Identity trust policy | ESO could not authenticate. Same class as leaving an IRSA annotation after migrating |
| Stale pins: 5 Helm charts, an Aurora engine version, a deprecated attribute | Only apply notices. Pinning is right for a database, but pins rot |

| Failure | Behaviour | Recovery |
| --- | --- | --- |
| Aurora writer dies | 25s of errors, promoted at ~50s | automatic, RPO 0 measured |
| AZ lost | one NAT and its nodes go | automatic, egress IPs pre-allowlisted |
| Valkey primary dies | Multi-AZ failover | automatic, `super_fetch` returns jobs |
| Spot reclaim | 2 minute warning to the interruption queue | automatic, critical never on spot |
| No capacity | pods Pending, existing capacity serves | widen instance families |
| Bad deploy | rollback under 2 minutes | Helm rollback or Git revert |
| Bad migration | old code runs against new schema | Backtrack if data destroyed |

The risk that worries me is organisational. Expand and contract works only if
every engineer follows it, and there is no CI check rejecting destructive DDL.
That is the one decision resting entirely on discipline.

## At real scale

- **State per component**, not per layer. Real blast radius separation and
  independent cadence, at the cost of an apply order a reviewer must follow.
- **Account per environment** in an Organization, state and deploy key in a
  tooling account.
- **Separate repositories** for modules, config and cluster state, with a
  promotion pipeline moving tested module versions dev to staging to production.
- **Private API endpoint** with self hosted runners in the VPC. Kept public with
  private access on and IAM access entries, because network position is the
  right control for a database and defence in depth for an IAM authenticated
  API. Tunnelling breaks TLS: the certificate SAN is the real EKS DNS name.
- **A CI check for destructive DDL**, the missing enforcement above.
