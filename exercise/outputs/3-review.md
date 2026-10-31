Agent: Claude Code, model Claude Sonnet 5.5 (`claude-sonnet-5-5`)

# ATAM-style review: ADR-0002 (Publish "order placed" event to RabbitMQ)

Files read: the ADR and `docs/scenario.md` only. I could not check the ADR's statements about `main.go`.

## 1. Quality-attribute scenarios

| # | Source · Stimulus · Environment | Response | Response measure |
|---|---|---|---|
| S1 Performance | Shopper · places order · 11.11 peak, 200 orders/s, MailFast slow (p99 8 s) | `PlaceOrder` finishes payment, shipping, publish | p95 ≤ 500 ms (today ≈ 300 ms, so ≈ 200 ms headroom for the publish, including the confirm wait) |
| S2 Availability | Broker, MailFast, `loyaltyservice` or analytics goes down · orders keep arriving · normal and peak load | Orders still succeed. Work queued before the outage completes after recovery | 0 orders failed by these components. 0 lost post-order events |
| S3 Reliability | Same `OrderPlaced` event delivered twice, or one lost · normal load | Points awarded once, then ledger unchanged | Points posted ≤ 1 min after the order, exactly once (no double points) |

## 2. Findings

| Type | Finding | Why it matters |
|---|---|---|
| Non-risk | Option 3 takes MailFast (p95 1.5 s, p99 8 s) off the order path. The email consumer limits itself to 50/s | Only a publish remains on the critical path. Meets S1 and respects the MailFast rate limit |
| Non-risk | Separate durable queue per consumer, retry + dead-letter queue for the ≈ 0.5 % MailFast failures | Failures stay inside the email consumer. A loyalty or analytics outage cannot delay emails (S2) |
| Non-risk | RabbitMQ over Kafka. The team has RabbitMQ experience, self-hosted, on Kubernetes, and replay is not required | Matches the team and operations constraints in the scenario |
| Risk | The ADR accepts losing the event when the publish fails after payment and shipping succeed. The scenario requires loyalty points awarded exactly once within 1 minute | A lost event means points are never awarded, which breaks S3. The ADR also lists "exactly once" as a positive consequence. The two statements contradict each other |
| Risk | The mitigation for the lost event (outbox) is deferred until the load test or production shows failures. It needs storage that `checkoutservice` "does not appear to have" | The outbox is the likely fix, but it would be a late, costly change to the order path. Silent loss only shows up as a customer complaint |
| Risk | Email backlog at peak: ≈ 150 emails/s over the limit. The ADR gives no peak duration or acceptable delay. For example, a 10-minute peak is ≈ 90,000 queued emails and ≈ 30 minutes to drain (my arithmetic, assuming a 10-minute peak, which the scenario does not state) | A confirmation email may arrive long after the order. The ADR does not decide whether that is acceptable |
| Risk | Duplicate emails: the ADR dedupes loyalty by `order_id` but not emails. A retry after a MailFast timeout can send a second email | The same at-least-once delivery that justifies loyalty de-duplication also affects emails. The ADR only lists this as an open question |
| Risk | The RabbitMQ cluster is a new single point of failure in the order path. Two of four developers know it | If the broker is down, orders still succeed, but all post-order work stops and events are lost (see first Risk). The ADR says "replicated cluster" without sizing |
| Sensitivity point | Publisher-confirm wait and retry time in `checkoutservice` | The headroom is ≈ 200 ms. A long or unbounded retry loop during a broker slowdown can push p95 above 500 ms. The ADR says "bounded time" without a number |
| Sensitivity point | Email consumer rate limit (≤ 50/s) and queue depth | Too high triggers MailFast rejections. Too low lengthens the backlog |
| Trade-off point | Asynchronous events: performance, availability and modifiability improve, but consistency, traceability and operational simplicity get worse | The ADR names these consequences, but gives no way to observe them (see Missing) |
| Trade-off point | Dropping the event on publish failure favours availability (order never fails) over reliability (points, emails guaranteed) | This is the central choice of the ADR, and it is deferred as an open question |

## 3. Missing or weak

- **Options not considered:** an outbox from the start (a variant of Option 3); a lighter durable queue; a managed broker (excluded by the stated preference, but worth one line). Option 2 is made weak by comparison: "goroutines" is the weakest in-process variant.
- **Unsupported claims:** "p95 stays near ≈ 300 ms plus one publish" is not measured. The Validation section only plans the test. Statements about `main.go` could not be checked here.
- **Inconsistency:** the header names a decider, but open question 1 asks who the deciders are.
- **Not stated:** the schema/versioning of `OrderPlaced`; the event carries the user's email (privacy, retention) and the analytics consumer's privacy rules are open; who owns the dead-letter queue and its alerts; how a trace id passes through the event.
- **Open questions that block the decision:** questions 2, 3 and 4 (peak duration, event loss, MailFast idempotency) affect the decision itself, not only the details.

## 4. Verdict

**Accept with changes.** Option 3 is the right direction, but the ADR leaves its own reliability requirement open.

1. **Settle event loss.** Either adopt the transactional outbox now, or state explicitly that points may be lost and get the business to sign that off. As written, S3 is violated.
2. **Add numbers:** a publisher-confirm timeout (for example a few tens of ms) inside the 200 ms headroom, and an agreed maximum email delay and 11.11 peak duration, with queue size and drain time.
3. **Cover duplicates beyond loyalty:** idempotency for emails (a MailFast idempotency key or a sent-log by `order_id`), and run the broker as a sized cluster with queue-depth, dead-letter and publish-failure alerts.
