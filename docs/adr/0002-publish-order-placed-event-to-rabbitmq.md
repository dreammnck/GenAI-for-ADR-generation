# ADR-0002: Publish an "order placed" event to RabbitMQ for post-order actions

- **Status:** Proposed
- **Date:** 2026-10-31
- **Deciders:** Nitchakran Chaipojjana (architect); approval by the architecture review board

## Context and problem

Requirement R-12 (`docs/scenario.md`) requires three new actions after an order is placed: (1) send the confirmation email through **MailFast**, replacing the mock `emailservice` logic; (2) award loyalty points through a new `loyaltyservice`; (3) record an "order placed" analytics event. Marketing expects more actions next year (SMS, fraud scoring, a recommendations update).

**How it works today** (`system/online-boutique/src/checkoutservice/main.go`, `PlaceOrder`): `checkoutservice` calls `chargeCard` (`paymentservice`), `shipOrder` (`shippingservice`) and `emptyUserCart`, then `sendOrderConfirmation` (`emailservice`), all **synchronously over gRPC** in sequence. If the email call fails, it only logs `log.Warnf("failed to send order confirmation...")` and still returns the order. `checkoutservice` holds one gRPC connection and one `*_SERVICE_ADDR` environment variable per downstream service (`emailSvcAddr`, `paymentSvcAddr`, …), so each new action would add another connection and another call inside `PlaceOrder`.

**Facts that constrain the decision** (`docs/scenario.md`, treated as given):

| Topic | Fact | What it means here |
|---|---|---|
| Checkout latency | p95 of `PlaceOrder` ≤ **500 ms**, today ≈ **300 ms** | Only ≈ 200 ms of headroom for everything new |
| MailFast | p95 **1.5 s**, p99 **8 s**, ≈ **0.5 %** failures, rate limit **50 emails/s** | A synchronous MailFast call alone exceeds the 500 ms budget at p95 |
| Peak load | **200 orders/s** (11.11 sale) | Orders arrive 4× faster than MailFast accepts emails (200 vs 50/s), so emails must queue at peak |
| Loyalty points | within **1 minute**, **exactly once** | Needs retry plus de-duplication, but allows asynchrony |
| Availability | an order must **never fail** because email, loyalty or analytics is down | These actions must be isolated from the order path |
| Team | 4 developers; strong in Go and gRPC; two have used **RabbitMQ** | |
| Operations | everything on Kubernetes; prefer **open-source, self-hosted** over cloud-only managed services | |

The problem: how should `checkoutservice` trigger these actions so that checkout latency and availability are protected, points are awarded exactly once within a minute, and next year's actions can be added cheaply?

## Decision drivers

1. **Availability:** an order never fails because email, loyalty or analytics is down.
2. **Performance:** `PlaceOrder` p95 ≤ 500 ms at 200 orders/s, although MailFast p95 is 1.5 s and p99 is 8 s.
3. **Reliability / correctness:** loyalty points exactly once and within 1 minute; MailFast's 0.5 % failures and 50 emails/s limit must be handled without losing emails.
4. **Modifiability:** add SMS, fraud scoring and recommendations update with little or no change to `checkoutservice`.
5. **Operability and cost:** fits the team's skills (Go, gRPC, some RabbitMQ), runs on Kubernetes, open-source and self-hosted.

## Considered options

1. **Synchronous gRPC calls** from `checkoutservice` to MailFast/`emailservice`, `loyaltyservice` and analytics, extending today's pattern
2. **In-process asynchronous calls**: `checkoutservice` starts goroutines (fire-and-forget) that call the same targets after the order succeeds
3. **Event publishing to RabbitMQ**: `checkoutservice` publishes one `OrderPlaced` event; `emailservice` (calling MailFast), `loyaltyservice` and an analytics consumer each have their own queue
4. **Event streaming with Kafka** (self-hosted): same event pattern, using a log-based broker

### Pros and cons of the options

| Option | Pros | Cons |
|---|---|---|
| 1. Synchronous gRPC | Simplest; same pattern the team already uses; no new component | MailFast p95 1.5 s breaks the 500 ms p95 budget; p99 8 s hurts the user; MailFast 50/s limit is hit at 200 orders/s and calls fail; every new action adds a connection and a call to `PlaceOrder` (low modifiability); latency is the sum of all calls |
| 2. In-process goroutines | No new component; removes latency from the response path; Go-friendly | Work is lost if the pod restarts or crashes; no buffering when 200 orders/s exceed 50 emails/s, so memory grows or calls are dropped; no durable retry, so "exactly once within 1 minute" for points is not assured; every new action still needs a code change in `checkoutservice` |
| 3. RabbitMQ pub/sub | Decouples `PlaceOrder` from all actions, so latency is one publish; durable queues buffer the 200 vs 50/s mismatch and survive consumer outages; retry and dead-letter queues handle the 0.5 % MailFast failures; a new consumer (SMS, fraud, recommendations) binds its own queue with no change to the producer; open-source, self-hosted, on Kubernetes; two developers know it | New stateful component to run (a single point of failure unless clustered); at-least-once delivery means duplicates are possible, so `loyaltyservice` must be idempotent; publish can fail or happen without the order being recorded (dual-write problem); eventual consistency; harder to trace across services |
| 4. Kafka | Durable log, replay of events, high throughput; suits analytics | The scenario names RabbitMQ experience on the team but none with Kafka; heavier to operate than the volume needs; replay is not a stated requirement; larger cost and learning curve for 4 developers |

## Decision

Choose **Option 3: publish an `OrderPlaced` event to self-hosted RabbitMQ**, and implement each post-order action as an independent consumer with its own durable queue.

- `checkoutservice` publishes the event at the point where it now calls `sendOrderConfirmation`, after payment and shipping succeed. A publish failure is logged and does not fail the order, as with email today.
- The event carries the `order_id` already generated in `PlaceOrder`, the user, the email and the order items.
- `emailservice` consumes its queue and calls MailFast, **rate-limiting itself to ≤ 50 emails/s** and retrying failures with backoff and a dead-letter queue.
- `loyaltyservice` consumes its queue and **de-duplicates by `order_id`** so redelivery cannot award points twice.
- The analytics consumer writes the "order placed" event.
- Next year's SMS, fraud scoring and recommendations update become new consumers bound to the same event.
- **Publish reliability:** `checkoutservice` waits for a publisher confirm and retries for a bounded time, then logs the failure and lets the order succeed. I accept that an event can be lost if the broker stays down past the retries; the publish-failure count is monitored. If it is non-zero in the 11.11 load test or in production, we adopt a transactional outbox (this needs new storage in `checkoutservice`).

Why this best satisfies the drivers: it takes MailFast's 1.5 s/8 s latency out of `PlaceOrder` (performance; Option 2 does too) and, unlike Option 2, keeps orders succeeding and work durable when a consumer or pod is down (availability), buffers the 200 vs 50/s mismatch with durable retry (reliability), and adds actions without touching the producer (modifiability). It matches the team's skills and the self-hosted, open-source, Kubernetes preference (operability), where Kafka does not. Option 2 is rejected because it has no durability or buffering.

## Consequences

- **Positive:** `PlaceOrder` p95 stays near today's ≈ 300 ms plus one publish, inside the 500 ms budget (quality attribute: performance).
- **Positive:** email, loyalty or analytics outages do not fail orders, and queued work is delivered when they recover (availability).
- **Positive:** points can be awarded within 1 minute with retries, and exactly once through idempotent handling (reliability).
- **Positive:** new post-order actions are added as consumers without changing `checkoutservice` (modifiability).
- **Negative:** RabbitMQ is a new stateful component to deploy, monitor and back up (operability, cost).
- **Negative:** emails and points are eventually consistent, and at peak the email queue grows because MailFast accepts 50/s against 200 orders/s (user-perceived latency of emails).
- **Negative:** tracing and debugging span several services (observability).
- **Risks and mitigations:**
  - *Email backlog at peak:* the queue grows by roughly 150 emails/s while orders exceed 50/s; monitor queue depth and set an alert, and settle the acceptable email delay (Open questions).
  - *Lost event* if the publish fails after payment and shipping succeed: use publisher confirms and retry; a transactional outbox may be needed, which requires a store `checkoutservice` does not appear to have (Open questions).
  - *Duplicate delivery:* idempotent consumers keyed by `order_id`; the loyalty ledger must enforce uniqueness.
  - *Broker outage:* run RabbitMQ as a replicated cluster with durable queues and persistent messages.
  - *MailFast failures (≈ 0.5 %):* retry with backoff, then dead-letter queue and alert.

## Validation

- **Latency test:** load-test `PlaceOrder` at 200 orders/s (e.g. with the Locust `loadgenerator`) and check p95 ≤ 500 ms.
- **Fault injection:** stop MailFast (or make it return errors), `loyaltyservice` and the analytics consumer one at a time and check that orders still succeed and that queued work completes after recovery.
- **Loyalty correctness:** replay the same `OrderPlaced` event several times and check that points are awarded exactly once; measure publish-to-award time against the 1-minute limit.
- **Rate limit:** check the email consumer never exceeds 50 emails/s and measure the queue drain time after a simulated 11.11 peak.
- **Modifiability:** add a stub consumer (e.g. SMS) and confirm no change is needed in `checkoutservice`.
- **Review date:** revisit this ADR when the first of next year's actions is scheduled, or if a step must run before another (e.g. fraud scoring before points), which could call for orchestration.

## Open questions

Not given in the context files, so not assumed:

1. **Duration of the 11.11 peak** and the acceptable delay for the confirmation email. The backlog (≈ 150 emails/s over the 50/s limit) cannot be sized without them.
2. **Publish reliability:** is losing an `OrderPlaced` event after payment and shipping acceptable? `main.go` shows no data store in `checkoutservice`, so a transactional outbox would need new storage; the team must decide.
3. **MailFast details:** API and authentication, cost, whether it supports idempotency keys to avoid duplicate emails on retry, and behaviour when the 50/s limit is exceeded.
4. **Loyalty rules:** how points are calculated, whether they are reversed on refund or cancellation, and where the points ledger is stored.
5. **Analytics:** the destination of the event, which fields it carries, and any privacy rules on personal data such as email.
6. **Future actions:** whether fraud scoring must complete before points or shipping (this would change the order of steps and may invalidate the pub/sub-only design).
7. **RabbitMQ sizing and HA:** cluster size, persistence settings and the team's operating capacity (two of four developers have used it).
8. **Fate of `emailservice`:** whether it is changed in place to call MailFast or replaced by a new service, and how the existing synchronous `EMAIL_SERVICE_ADDR` call is retired.
