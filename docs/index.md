# Platform Engineering Exercise

This site holds the design document and the runbook for a proprietary trading
platform built on AWS. The Terraform that implements it lives in a separate
private repository.

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
were rejected, what the platform costs and how it fails. The
[runbook](runbook.md) covers deploying, rolling back, and one failure scenario
worked through end to end.
