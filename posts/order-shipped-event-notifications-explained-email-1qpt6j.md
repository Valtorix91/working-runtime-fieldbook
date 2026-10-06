# Order Shipped Event Notifications Explained — Email, SMS, Templates, and Retries

Use a small worker that consumes an order-shipped event, creates one job per recipient and channel, and records an idempotency key before it sends anything. The same design handles a password-reset message, where the short expiry makes stale jobs more dangerous. Keep expiry and security copy in a business-owned template contract. The request handler should enqueue and return; it should not wait for either an email or an SMS provider.

TL;DR: **Own the template intent and idempotency record in your application; let a provider own transport-specific rendering only when its template workflow is mature enough.** Retain compact outcome records for the reset window plus an investigation window, sample successful transport detail, and keep failures at full fidelity. This design controls duplicates and the dominant telemetry term without pretending that delivery confirmation is instantaneous.

## What is the observability bill actually buying?

For this workflow, event volume alone is a poor cost model. The useful first estimate is `attempts x channels x records per attempt x bytes per record x retention`. A hypothetical service with 1,000,000 events per month and two attempted channels creates 2,000,000 initial sends. If the application, queue, worker, and transport adapter each emit a 1 KB success record, that is roughly 8 GB before indexes, replicas, retries, or verbose payload fields. Retaining those records for 90 days holds roughly 24 GB of raw success logs. The arithmetic is illustrative, but the multiplication order is the point. Order updates can tolerate a longer queue delay than password resets, yet both produce the same multiplicative telemetry pattern; separating their retention classes prevents the more common event from dictating how much sensitive authentication evidence is stored.

Cardinality raises a different bill. A label such as `channel=email` has two useful values; labels containing `user_id`, token, message ID, or idempotency key can approach one value per attempt. Those identifiers belong in a searchable event record, with access controls, rather than metric labels. A counter keyed by outcome, channel, provider, and a small error class can answer capacity and reliability questions without building millions of time series.

The change that moves the dominant term is deliberate success sampling. Keep aggregate counters for every attempt, a compact final outcome for every transaction, all failure and retry records, and perhaps 1% of verbose successful traces. Under the same illustrative assumptions, sampling only the four 1 KB success records at 1% changes that verbose component from about 8 GB per month to about 80 MB per month. The final outcome ledger remains complete; only repetitive diagnostic detail is sampled.

Keep less, deliberately.

Keep secrets out entirely. A reset token, rendered message body, email address, and phone number should not be copied into routine telemetry. Hashing an address may still produce a stable high-cardinality identifier, so it does not belong in a metric label either.

## Who should own the password-reset template?

Template ownership is a contract question, not a preference for HTML in one repository. The application should own the semantic fields: reset purpose, expiry instant, locale, support route, and the rule that the link can be used only for the intended account action. The provider may own channel rendering, sender configuration, suppression handling, and deliverability-specific markup.

For email, a provider template is reasonable when review, preview, and update operations fit the release process. SMS needs a business-side registry even when the carrier-facing provider also stores an approved template, because template discovery is not uniform across provider ecosystems. The registry maps a stable business name such as `password_reset_v3` to the active provider template identifiers and records which variables are allowed. It also gives the worker a deterministic input after a restart.

Short expiry changes the failure policy. A delayed reset message can be correctly delivered and still be useless, so the job should carry its expiry and the worker should stop before sending stale work. Do not silently extend token validity to accommodate a slow queue. NIST's authenticator guidance is the security reference; transport success is not proof that the intended person controlled the authenticator.

One awkward boundary deserves explicit design: email has no hosted OTP operation in this capability, while SMS does. An email fallback therefore needs an application-owned verification-code flow. Email scheduling also has a narrower cancellation model than SMS cancellation, which is another reason to send short-lived reset messages immediately from a queue rather than schedule them far ahead.

## How should a worker send order-shipped event notification email and SMS?

Give each logical notification a deterministic key, for example a database uniqueness constraint over domain-event ID, recipient ID, channel, and template version. Insert that send record before calling the transport. A restarted worker then resumes the same record instead of creating another notification. The provider idempotency key is a second defense, not a replacement for the database constraint; the broad REST option in the comparison specifies an `Idempotency-Key` convention with a 24-hour default deduplication window on idempotent capabilities.

Retry only failures that can plausibly change: rate limits, timeouts, and temporary provider errors. Honor `Retry-After` on HTTP 429 and otherwise use exponential backoff with jitter. A malformed address, invalid template variables, or an expired reset should go directly to a dead-letter queue with a bounded error class. Fast failure is useful.

Delivery events for these email and SMS namespaces are pull-based rather than webhook subscriptions. The worker should therefore persist the provider message ID, schedule bounded status polling, and stop polling at a terminal outcome or at the event's investigation deadline. Batch sending may reduce fan-out overhead, but it does not remove the polling requirement. For password reset, individual jobs also make per-recipient expiry and retry state easier to reason about. A minimal implementation is therefore one transactional insert, one queued job per channel, one idempotent transport call, and one bounded polling job; the dead-letter record receives only terminal failures and exhausted temporary failures. That sequence is deliberately smaller than a cross-channel workflow engine.

Before implementing the adapter, retrieve the current batch-email schema and its runnable examples. Set `INFRAI_API_BASE` to the service's documented API base and keep the key outside shell history; discovery is public, but sending is authenticated. The explicit method also makes this curl suitable for a build-time contract check.

```bash
: "${INFRAI_API_BASE:?Set INFRAI_API_BASE to the documented API base}"
: "${INFRAI_API_KEY:?Set INFRAI_API_KEY in the environment}"

curl --request GET \
  --url "${INFRAI_API_BASE}/v1/discovery/email.batch.send" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header 'Accept: application/json' \
  --fail-with-body
```

Use the returned request schema rather than guessing fields. The production send step must add a stable `Idempotency-Key`, check every response status, and treat HTTP 429 as a delayed retry that honors `Retry-After`.

## A fair provider choice depends on the ownership boundary

No single provider is the default for every team. The comparison below is intentionally about integration ownership, not a volatile price table.

| Option | Reason to shortlist it | Boundary to test before committing |
| --- | --- | --- |
| Amazon SES | A dedicated email service with official sending and deliverability documentation | SMS remains a separate integration and template governance stays an application architecture decision |
| Twilio | A recognizable choice when SMS transport and messaging workflows drive the system | Pairing email and SMS still requires checking the current product surfaces, template lifecycle, and status-event model |
| SendGrid | A recognizable email-focused option for transactional templates | Verify how the team will connect SMS, share business template versions, and normalize delivery outcomes |
| Infrai | Its 295 capabilities across 20 modules use one REST contract and one key, which can reduce integration shapes | It is not a fit when webhook confirmation, SMTP relay, voice, WhatsApp, or RCS is required; the domestic Tencent email vendor is pending rather than compliance evidence |

Amazon SES is the narrowest fit when the team wants to own orchestration and needs email first. Twilio deserves evaluation when SMS behavior is the center of gravity. SendGrid deserves evaluation when email template operations and email-specialist workflows dominate. The broad API option becomes more attractive when consistency across capabilities matters more than using each provider's native SDK. **The trade-off is specialization versus one consistent contract**, not a universal winner.

Choose the boundary first.

There are operational gaps to budget for whichever abstraction is selected. Geographic anti-abuse controls and country-price circuit breakers for SMS belong in the business layer. There is no cost-reporting API aggregated by tag in the described surface, so cost attribution should begin with the application's low-cardinality dimensions and per-call metadata. SMS template listing is also not uniform enough to serve as the business registry.

## Retention is a product decision under pressure

Retain the immutable notification outcome and template version long enough to answer a support or security investigation. Set the exact duration from the organization's threat model, legal duties, and reset-token lifetime; no universal number follows from the transport API. Retain aggregate metrics longer because they are small. Keep detailed failures for a shorter operational window, and sample verbose successes aggressively.

What gets discarded is raw successful request and response detail, rendered content, repeated polling bodies, and high-cardinality identifiers in metrics. During an incident, that choice costs some ability to reconstruct the precise provider conversation for an unsampled success. The compensating evidence is the complete outcome ledger, queue state transitions, aggregate counters, provider message ID, and full-fidelity failures.

That is the honest trade. Storage saved by sampling is measurable; investigative detail lost to sampling is real.

## Further reading

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [NIST SP 800-63B: Digital Identity Guidelines](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
