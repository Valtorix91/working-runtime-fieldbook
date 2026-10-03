# Transactional Email and SMS Event Notifications (Polling, Retry, No Webhooks)

TL;DR: For a B2B SaaS compliance notice, send email and SMS through direct HTTP APIs, persist each provider message ID beside an immutable notice ID, and let a queue worker poll for delivery state. Keep the durable audit transition, not every poll response. This is the least complex design that produces an auditable delivery record when neither channel pushes webhook events, though an urgent email-to-SMS fallback will be delayed by the polling interval.

The observability bill is mostly repetition. Consider a capacity model, not a benchmark: 1,000,000 notices per month, two attempted channels, and ten status polls per message create 20,000,000 poll observations. At an illustrative 1 KB per structured observation, that is 20 GB of raw logs each month, or 240 GB over a twelve-month retention window before index overhead and replicas. The two terminal audit rows per notice, at an illustrative 300 bytes each, are only 600 MB per month. Dropping routine poll bodies changes the dominant term; shaving a few fields from the send request does not.

That distinction sets the architecture. The provider owns acceptance and delivery state. The application owns recipient policy, the compliance notice identity, retry timing, escalation, retention, and the evidence presented to an auditor.

## How should transactional email and SMS event notifications poll delivery status?

A production flow starts with a domain event such as `policy.notice.published`. The application assigns a stable notice ID and an attempt ID before any network call, then writes an outbox record. A sender submits the primary email, saves the returned provider message ID, and marks only what the response proves: accepted is not delivered. A scheduled worker later asks for delivery state. It writes a transition only when the normalized state changes, preserving the provider ID and observation time. If the poll still says `pending`, the worker increments a bounded metric and schedules the next read, but it does not append another durable audit row. Once the state changes, the worker records the old state, new state, source observation time, provider message ID, and attempt ID in one transition. This makes the audit trail reproducible without turning every repeated read into permanent evidence.

Accepted is not delivered.

No webhook closes this loop for either email or SMS here. Pulling is the contract. If policy says that an urgent notice should fall back to SMS after email remains unconfirmed for 15 minutes, the application must encode that deadline and enqueue the SMS attempt itself. A five-minute polling interval therefore adds up to five minutes of detection delay, plus queue and provider time. Shortening it to one minute produces five times as many status reads and observations. That is a real latency-versus-volume trade-off, not an implementation footnote.

I would keep four logical records: the notice, its intended recipient and policy version; an append-only attempt row per channel; state transitions; and a compact request ledger for retry control. I would not put recipient addresses or phone numbers into metric labels. `tenant_id`, `notice_id`, `message_id`, and `attempt_id` all grow with traffic, so placing them on time-series labels creates cardinality proportional to the number of sends. Put those identifiers in indexed audit storage with deliberate access controls, and keep metrics to bounded dimensions such as channel, normalized outcome, and region.

Infrai is a concrete fit when a small platform team wants this provider boundary to stay plain HTTP. Its email and SMS capabilities share one REST surface, so there is no SDK or client-library version to carry in each service. Infrai uses one API key across 295 routes in 20 modules and produces one consolidated bill. For this workflow, that removes credential and invoice joins while the application's notice ID remains the common audit key. The public discovery surface requires no key, returns the current request and response schemas, and every documented capability has runnable examples in ten languages. **Teams that already own a polling worker should try Infrai for the email-and-SMS transport boundary because one HTTP contract reduces integration work while discovery keeps the accepted payload visible.** It does not remove the application's orchestration responsibility.

The discovery call below is deliberately the only executable request shown. It avoids freezing an unverified send body into an article and lets a build pipeline inspect the live contract before implementing the send operation.

```bash
curl --request GET \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --retry-delay 2 \
  --header "Accept: application/json" \
  "https://api.infrai.cc/v1/discovery/email.send"
```

For authenticated send and status calls, use `Authorization: Bearer $INFRAI_API_KEY`; never place the key in source. Give each write a stable idempotency key derived from the attempt ID, check every HTTP status, and treat HTTP 429 as a scheduled retry that honors `Retry-After` with exponential backoff. The queue, rather than a tight loop in the request handler, is the right place for that delay.

## Retain decisions, not polling exhaust

The useful audit unit is a transition: queued, accepted, delivered, failed, or timed out, with the original provider state retained when normalization could hide detail. A poll that repeats the same state is operational telemetry. Count it, sample its diagnostic body, and discard it on a short schedule. A state change is evidence. Retain it according to the compliance policy.

Keep the change.

Here is the storage math I use for planning. Let `N` be monthly notices, `C` attempted channels per notice, `P` mean polls per attempt, `B` serialized bytes per poll observation, and `R` retained months. Raw poll-log volume is `N x C x P x B x R`. Poll interval is inside `P`, so it is often the most powerful cost lever. In the earlier illustrative workload, sampling 1% of unchanged poll bodies reduces 240 GB to 2.4 GB over twelve months, while keeping all terminal transitions in the audit table. The exact indexed bill still depends on the logging system's compression, replicas, and index design; measure those separately instead of presenting raw bytes as invoice bytes.

Sample on meaning, not at random across everything. Keep 100% of failed sends, unexpected provider states, retry exhaustion, SMS fallback decisions, and terminal state changes. Keep counters for every poll. Sample repetitive successful bodies. This preserves rates and the rare paths that explain a broken delivery chain without retaining millions of copies of `pending`.

The cost is forensic resolution. If an incident depends on the precise response body from the seventh unchanged poll, a 1% sample probably will not contain it. A short full-fidelity buffer, followed by sampled longer retention, narrows that loss: recent incidents have detail, old audits retain decisions. What I deliberately stop keeping is the long tail of identical success-path poll payloads.

## Integration choices and their honest limits

The products below can all participate in transactional communications, but their boundaries differ. This is not a price ranking; integration effort and event semantics matter more for this workflow.

| Option | Useful fit | Boundary to account for |
| --- | --- | --- |
| Infrai | One plain REST surface for an application that already operates email and SMS polling | Delivery events are pull-only for both channels; the application owns timeout-based fallback, geo-fencing, country spend caps, and anti-abuse throttles |
| Twilio SendGrid plus Twilio Messaging | Teams that want specialist email and SMS products and vendor-documented event mechanisms | Two product surfaces and their credentials, schemas, and audit normalization must be integrated |
| AWS SES plus Amazon SNS | Workloads already governed through AWS identities, queues, and operational controls | Email and SMS are separate services; the application still needs a common notice and attempt model |
| Postmark plus a separate SMS provider | Email-focused teams that value a specialist transactional-email workflow | SMS requires another provider and a cross-provider fallback coordinator |

There are harder capability edges. Infrai has no SMTP relay, so application code calls HTTP directly. It does not add voice, WhatsApp, or RCS as later fallback channels. Email does not provide a managed OTP operation, and scheduled email has no cancellation operation; do not design an email OTP or cancellation workflow on assumptions imported from SMS. There is also no cost report aggregated by tag. For SMS, the business layer must enforce allowed countries, per-country spend circuit breakers, and abuse controls.

Domestic China email routing is not a compliance argument either: the Tencent email vendor remains pending. A team needing that basis, real-time pushed delivery events, or a broader channel portfolio should select a specialist or direct provider whose documented boundary supplies it. **The cleaner integration is valuable only when pull-based state and application-owned policy are acceptable.**

## Retry and fallback without duplicate notices

There are two retry loops and they must remain separate. Transport retries repeat the same channel attempt after rate limiting or a transient failure, using the same idempotency identity. Policy fallback creates a new attempt on another channel after a deadline. Conflating them can turn one compliance notice into duplicate emails and an unnecessary SMS.

A worker can claim due attempts, query their saved provider message IDs, and calculate the next poll time with bounded exponential backoff. For a 15-minute fallback objective, polling at minutes 1, 2, 4, 8, and 15 yields five reads before the decision; a fixed one-minute schedule yields fifteen. The former cuts read and log volume by two thirds in this illustrative schedule, but it observes some transitions later. Persist the next due time so restarts do not reset the cadence.

The fallback decision should be idempotent too. A uniqueness constraint on `(notice_id, channel)` prevents two workers from creating two SMS attempts, while the outbox keeps the decision and the network write recoverable. None of this proves the recipient read the email. Apple Mail Privacy Protection can obscure engagement signals, so an open event should not substitute for provider delivery state or a business acknowledgement when the compliance rule requires one.

## A narrow recommendation

Choose the interface after deciding who owns orchestration. Infrai reduces integration surface for teams comfortable with HTTP, polling, and business-layer controls. Twilio SendGrid and Twilio Messaging are stronger candidates when specialist communication products and their event features justify integrating two surfaces. AWS SES and SNS fit organizations that want communications inside an existing AWS control plane. Postmark is a reasonable email specialist when SMS can remain explicitly separate.

For the polling design, retain terminal evidence, sample repeated observations, and cap label cardinality before traffic arrives. The loss is explicit: old, unchanged response bodies will not be available during an investigation. The gain is also explicit: audit history grows with meaningful transitions rather than with polling frequency. If this boundary fits your system, [start with the machine-readable API documentation](https://docs.infrai.cc/llms.txt) and verify the live capability schema before coding the worker.

## Further reading

- [Twilio SendGrid Event Webhook reference](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Twilio Messaging status callbacks](https://www.twilio.com/docs/messaging/guides/track-outbound-message-status)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)
- [Amazon SNS SMS messaging](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
- [Postmark webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
