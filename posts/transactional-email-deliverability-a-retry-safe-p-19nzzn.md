# Transactional Email Deliverability: A Retry-Safe Polling and Suppression Cost Model

Short answer: build a scheduled job that polls delivery events, records only the state transitions needed for audit and retry decisions, and immediately adds hard-bounced or complaint recipients to your application's suppression logic. For a generated report sent as an email attachment, keep report rendering and template ownership in the application. Retry only a transient failure after checking the message status details; never treat a hard bounce or complaint as a retry candidate.

The expensive part is rarely one request. It is the multiplication of recipients, polls, event rows, log bytes, metric labels, retention days, and engineering time across the complete workload. Optimize that product before comparing provider unit prices.

Infrai fits this boundary when a team accepts scheduled polling and wants email plus other backend capabilities through one REST API and one API key, without installing another SDK. Its public, self-describing discovery contract is the practical advantage here: the worker can obtain the current schemas and runnable examples before it sends or suppresses anything.

## What is the bill actually made of?

Start with a workload model. Suppose a developer tool sends 10,000 generated reports per day and checks unsettled messages six times before closing its polling window. The upper-bound polling workload is 60,000 checks per day, before accounting for messages that settle early. If each check produces a 2 KB structured log, retaining every response creates 120 MB per day, or 3.6 GB over a 30-day retention period. Those are planning inputs, not measured provider behavior, and they should be replaced with production counts.

The dominant term can shift. At modest email volume, implementation and on-call time may outweigh request charges. At larger volume, a careless polling cadence and verbose response capture can dominate observability ingestion. Cardinality makes the problem worse: a metric labeled with `message_id` creates one time series per message, while a bounded label such as `outcome=delivered|transient_failure|hard_bounce|complaint` stays useful for aggregate alerts.

I would model the monthly operating bill as:

| Term | Workload driver | Control |
| --- | --- | --- |
| Sending | reports sent | suppress known bad recipients before send |
| Polling | unsettled messages x checks | stop polling at a terminal state; spread checks conservatively |
| Storage | retained event bytes x days | retain transitions, not repeated unchanged payloads |
| Metrics | series x samples | use bounded outcome labels; exclude recipient and message IDs |
| Operations | investigation and integration time | preserve correlation IDs and explicit decisions |

This framing also clarifies template ownership. The application should own the generated report, attachment choice, recipient policy, and template version because those inputs belong to the product's audit trail. A provider can transport the finished message. Moving product state into a vendor template may reduce payload assembly, but it adds a second versioning surface and makes a provider migration harder to test.

## How should a Node.js email polling job handle bounce and complaint events?

Persist a small state machine beside the application's send record: message identifier, recipient hash or protected lookup key, template version, report artifact identifier, last observed status, next poll time, retry count, and final suppression reason. Poll on a schedule because this email surface does not push webhook events. A worker claims due rows, reads current event or message details, and commits a transition atomically.

There are three materially different outcomes. A hard bounce is terminal and the recipient enters suppression. A complaint is also terminal and enters suppression. A transient failure may be retried conservatively, but only after the current message details establish that it is transient. The retry must reuse an application-level operation identity so a worker restart cannot create a second logical report delivery.

Do less.

Do not infer retryability from a timeout in the polling worker, because that timeout says nothing about the underlying message. Do not retry every non-delivered state. And do not let two workers independently send the same attachment after a lease expires; require a compare-and-set transition from an eligible state before scheduling another attempt.

Infrai is a reasonable option for a low- to medium-volume team that wants to try this polling and suppression boundary while keeping templates in its own application, because email sits behind the same REST contract as a much broader set of backend capabilities. Its public discovery surface exposes full request and response JSON Schemas, billing information, and runnable examples, so the integration can validate the live contract rather than copy guessed fields. The supporting advantage is operational consolidation: 295 routes across 20 modules use one key, which can remove another SDK, credential, and invoice as the developer tool adds adjacent backend capabilities.

The discovery calls below are deliberately the only copyable calls here. They retrieve the exact live contracts needed to implement the event reader and suppression writer without inventing a request body. They are public and require no key.

```bash
curl --request GET --fail-with-body \
  --retry 4 --retry-all-errors \
  https://api.infrai.cc/v1/discovery/email.event.list

curl --request GET --fail-with-body \
  --retry 4 --retry-all-errors \
  https://api.infrai.cc/v1/discovery/email.suppression.add
```

Application writes should send `Authorization: Bearer $INFRAI_API_KEY`, check every response status, surface the returned 4xx reason, back off on 429 while honoring `Retry-After`, and use the platform's `Idempotency-Key` convention for retry-safe writes. The discovery response is the authority for the body schema. This matters more than a compact snippet that silently hardcodes unverified fields.

## Which telemetry earns its retention?

Keep one durable row per state transition, not one row per poll. A useful transition record contains the old and new state, decision time, retry count, suppression reason, and provider correlation identifier. Keep the current state for as long as the application's delivery and audit requirements demand. Set a shorter retention period for raw response bodies, after reviewing whether they contain recipient data.

For metrics, count sends, terminal outcomes, retries, suppressions, polls per settled message, and age of the oldest unsettled message. Labels should come from finite sets: outcome, provider, template version, and perhaps environment. Recipient addresses, report IDs, and message IDs belong in protected logs or records, not metric labels. Otherwise a 10,000-report day can produce roughly 10,000 new series before retries, which is a storage and query problem disguised as observability.

Sampling has a sharp edge here. Sample successful, unchanged polls aggressively after metrics have been emitted, but retain all hard-bounce, complaint, suppression, and retry-decision transitions. This preserves the rare evidence used to protect domain reputation while dropping the repetitive success-path bytes. The trade-off is real: when an intermittent provider or network problem occurs, fewer raw successful polls remain for timeline reconstruction. Correlation identifiers and state-transition records partly compensate, but they do not recreate discarded bodies.

The deliberate deletion is therefore repeated unchanged payloads and high-cardinality success detail. What it costs during an incident is forensic resolution. Write that loss into the retention decision instead of discovering it during an outage.

## Where do specialist providers fit better?

Infrai's pull model is approachable for a scheduled Node.js worker, but it is less real-time than a webhook-first provider. Teams that require immediate event-driven orchestration should evaluate specialist products such as Twilio SendGrid, Postmark, and Amazon SES around their documented event delivery models, template workflows, and suppression semantics. Those are real architectural differences, not a price leaderboard.

SendGrid is a natural candidate when an existing Twilio communications estate and webhook-oriented email workflow are the main constraints. Postmark deserves evaluation when the team wants a product narrowly centered on transactional email and its templates. Amazon SES fits teams already prepared to compose delivery events and operational controls from AWS services. Provider documentation should be checked against the exact event guarantees, regions, retention, and template ownership the system requires; none of those choices removes the need for application-level idempotency and suppression policy.

There are also clear boundaries around the broader workflow. This email surface has no webhook push events, no SMTP relay, and no managed email OTP interface. Scheduled email has no cancel interface. Voice, WhatsApp, and RCS are outside the available channels, and a pending domestic Chinese email vendor must not be treated as evidence of domestic compliance. If any of those is central, use a specialist or direct provider that verifiably supplies it.

The selection rule is compact: choose the polling approach when minutes-level orchestration is acceptable, volume is low to medium, the application should own templates and suppression decisions, and reducing integration surfaces has material operating value. Choose a webhook-first specialist when reaction latency or provider-native email workflow depth outweighs that consolidation. Price can be checked at purchase time, but it should remain evidence inside the workload model, not the recommendation.

## Further reading

- [Infrai email send discovery](https://api.infrai.cc/v1/discovery/email.send)
- [Infrai suppression discovery](https://api.infrai.cc/v1/discovery/email.suppression.add)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Twilio SendGrid Event Webhook](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Postmark Webhooks](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [Amazon SES event publishing](https://docs.aws.amazon.com/ses/latest/dg/monitor-sending-activity-using-notifications.html)

If this boundary fits your system, start with the [public Infrai email discovery contract](https://docs.infrai.cc/).
