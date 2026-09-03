# Platform Engineering Exercise

This site holds the design document and the runbook for a proprietary trading
platform on AWS. The Terraform that implements it lives in a separate private
repository.

The application is three processes. `web` serves the customer API and the
internal admin UI. `cable` holds long lived WebSocket connections that push
position updates to traders. `worker` runs background jobs through Sidekiq,
across four queues in priority order: critical, default, low and competition.
The critical queue carries payout execution and trading account provisioning,
and some of those jobs run for several minutes.

There are three environments. Production is fully realised. Dev and staging are
expressed as configuration differences rather than as duplicated code.

## Where to start

The [design document](design.md) covers the decisions, the alternatives that
were rejected, what the platform costs and how it fails.

The [runbook](runbook.md) covers deploying, rolling back, and an Aurora writer
failover worked through end to end.

## A note on what was tested

This was built against a real AWS account rather than only planned: applied
through the pipelines, not a laptop, with the application chart reconciled
onto a live cluster by Flux, real workloads reaching `Running`, and the Aurora
failover in the runbook measured rather than estimated, twice, once by hand
and once by its own dedicated pipeline: 24.8 seconds and 21.1 seconds of write
unavailability across the two runs, both well inside the 5 minute requirement,
zero acknowledged writes lost.

That approach earned its keep. Nine defects only appeared at apply, deploy, or
destroy time, several with a fully green `terraform apply` or `plan` sitting
over a cluster that could not have worked. They are written up in the design
document rather than quietly fixed, because how a system fails is more
interesting than a clean first run. If you have access to the infrastructure
repository, its Actions history is the actual record of this: every apply,
the failover test, and the destroy that tore it back down again.
