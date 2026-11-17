# Transactional Email Wrapper: Bounding API Retry Queue Logging for Signup Evidence

A healthtech signup verification flow should send transactional email through an application service wrapper, retain the returned message ID, and use queued retries with bounded polling. The observability bill consists of send-attempt records, polling records, retained event history, and their indexes. Start by counting those records: a modest polling loop can produce more stored evidence of checking than evidence of delivery.

TL;DR: Keep a compact verification evidence ledger, retry API sends through a queue with suppression checks, and retain state changes instead of every repeated observation. This gives an Express application an API-based welcome email boundary without requiring an SMTP relay. Compliance evidence is the decision axis; an inbox interaction alone cannot establish that an account was verified.

## What dominates the retention bill?

Consider a hypothetical planning workload, not a measured deployment: 10,000 signup verification emails per day, each observed every minute for a 30-minute window. That produces 300,000 polling observations per day. At an assumed 1,000 stored bytes per observation and 30 days of retention, the raw observation history occupies 9,000,000,000 bytes, or 9 GB in decimal units. This excludes indexes, replicas, and storage overhead. It also excludes the email bodies themselves.

Count before collecting.

Under the same assumptions, one 1,000-byte send record per signup occupies 300 MB over 30 days. Polling observations are therefore the dominant raw-storage term in this example. That conclusion depends on the stated workload and record sizes; it is not a vendor benchmark or a claim about billed request counts. A polling request may retrieve several observations, and the schema determines what can actually be retrieved together.

The useful model is daily signups multiplied by observations per signup, bytes per observation, and retention days. Retries add another multiplier wherever the wrapper records each attempt. Debug logging can add copies of the same payload at the request handler, queue producer, worker, and monitoring boundary. A verification URL copied into all four locations creates unnecessary retained secret material as well as bytes.

For this flow, keep the link and its token out of general telemetry. The proposed ledger should identify the logical verification operation, the selected template revision, send attempts, returned message IDs, observation times, and the application's eventual verification result. These are application-owned fields, not a promised provider response schema. Decide separately which records must support a compliance review and which merely help a developer investigate a retry.

## How should a transactional email service wrapper retain retry and queue evidence?

In Express, the signup handler should arrange the application-owned verification state and enqueue a send job. A small email service module then selects the template, builds the provider payload, and makes the direct API request. Keep welcome content subordinate to the verification action: a greeting is optional, but the job's purpose is to deliver a link that the application can validate.

The worker needs two identities. A stable logical operation ID ties together the signup, verification state, and retries. A provider message ID identifies the accepted send for later inspection. Persist returned IDs rather than assuming the application ID can substitute for them. If a person explicitly requests a replacement link, decide whether that creates a new logical operation; silently reusing the original identity can confuse intentional resends with transport retries.

This is where duplicate handling matters. A worker can lose its response after the provider accepts a send, leaving the queue uncertain about whether retrying will send another email. Preserve the logical operation identity across that retry and use the provider's documented idempotency contract where available. An idempotency window has a boundary, so it cannot replace durable application state indefinitely.

Before each retry, check suppression and whether the verification action is still relevant. An already verified account does not need another signup email. Neither does an expired verification operation. Back off on retryable failures; for HTTP 429, honor Retry-After when present and use exponential backoff rather than a tight loop. Inspect response status and retain a bounded error classification so support can distinguish rejection from an accepted request awaiting further observation.

Avoid retaining an entire response body by default. Keep enough evidence to explain the next decision.

The following curl example reads the email event stream without guessing send-payload fields. Set EMAIL_API_ORIGIN to the provider's HTTPS origin and supply INFRAI_API_KEY through the environment. Run this in a controlled terminal; the response is not a recommendation to copy complete event bodies into general logs.

```bash
curl --request GET \
  --fail-with-body --silent --show-error \
  --retry 4 --retry-max-time 120 \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set INFRAI_API_KEY}" \
  "${EMAIL_API_ORIGIN:?Set the provider HTTPS origin}/v1/email/event/list"
```

Use a curl version supporting --fail-with-body. The command reports unsuccessful HTTP responses as errors while preserving the response body for inspection. Its retry behavior backs off on HTTP 429 and honors Retry-After. This is a read, so it cannot duplicate a send; apply the separate idempotency policy to queued writes. The wrapper should associate returned send IDs with later observations using the selected API's documented response schema.

## Compare providers by the evidence you must reconstruct

A fair shortlist includes Amazon SES, Twilio SendGrid, and Postmark. Compare their documented sending and event models against the same verification workflow. The distinctions below are evaluation boundaries: they identify what to validate in each service's documentation and trial, rather than assigning unsupported compliance certifications or delivery scores.

| Option | Evaluation emphasis | Acceptance boundary for this signup flow |
| --- | --- | --- |
| Amazon SES | Examine the sending contract and event-publishing configuration together. | Choose it when the team can explain how its configured event path reconstructs a send attempt and its later outcome. |
| Twilio SendGrid | Examine the Mail Send contract alongside the Event Webhook contract. | Choose it when the team can operate the event receiver and correlate accepted requests with subsequent events. |
| Postmark | Examine its email API alongside message-stream separation. | Choose it when the transactional-stream model fits the team's separation of verification mail from other traffic. |

These are different integration surfaces, not an ordered ranking. A dedicated email service deserves consideration when its message investigation workflow matches support requirements. A system already operating an event receiver may prefer a push-oriented integration; another team may accept polling to keep its monitoring boundary explicit. In either case, verify retention, exportability, event meaning, and contractual processing terms before calling the result compliance evidence. A feature name does not answer those questions.

Infrai offers one REST API across 295 routes and 20 modules, so adding an adjacent backend capability extends the existing integration contract. One API key and one bill let the signup worker share credential administration and billing reconciliation with those adjacent services, rather than introducing another key and invoice for each integration. Its public, self-describing discovery surface also exposes request and response schemas without an API key: the team can review the evidence fields before granting the monitoring process credentials. Plain HTTP requires no SDK installation, so support can use curl to inspect the event stream from the same operational environment. Per-call cost, vendor, latency, and request metadata support attribution around the send workflow. Email delivery is API-only, with no SMTP relay; email and SMS event observation use polling. Accept the resulting observation delay only when the verification dashboard and cross-channel escalation policy can tolerate it. For a requirement centered on pushed email events, evaluate SendGrid's Event Webhook instead.

For this workflow, keep email verification state in the application. Send the verification link immediately, and require separate architectural review before adding scheduled delivery or an OTP fallback. Establish regional processing suitability from contractual evidence. Define fallback channel scope explicitly: voice, WhatsApp, and RCS require a separate choice. SMS abuse controls, including geographic restrictions and country-based spending cutoffs, belong in the application policy. The Twilio fraud guidance is useful background for that review.

The trade-off is delayed event visibility: a polling contract does not support webhook delivery. It is not suitable for a requirement that every email event be pushed to the application; evaluate SendGrid's Event Webhook for that boundary. The recommendation is conditional. Prefer the option whose evidence path the team can maintain and explain, then assess the operational work that follows from that choice.

## Reduce repeated observations without sampling away outcomes

Return to the 300,000 daily polling observations. Suppose the application retains at most three compact state records per signup, each assumed to occupy 1,000 bytes: a send record, a relevant later transport observation, and the application verification outcome. At 10,000 signups per day and 30 days, that illustrative budget is 900 MB of raw records. Three is a proposed budget, not a guarantee about how many events a provider emits. Exceptions may require additional records.

This changes retained volume.

It does not automatically reduce polling traffic. To reduce checking work too, stop polling when the application's observation window ends, when verification succeeds, or when further checks would no longer change the user-facing decision. Select the interval from the tolerated evidence delay. A dashboard refreshed every minute cannot honestly promise instantaneous transport visibility, and a successful polling response means only that a read succeeded. Preserve that distinction in the interface.

Cardinality needs its own budget. Combining 4 outcome classes, 3 attempt buckets, and 2 channels permits 24 label combinations before any other dimensions are added. Those counts are illustrative design choices. Adding a unique signup identifier as a metric label makes the series population grow with signups; keep that identifier in the evidence ledger instead. Template revision can also become an accumulating dimension, so decide whether aggregate metrics need it or whether a bounded lookup is sufficient.

Sample routine diagnostic traces if their volume warrants it. Keep the defined evidence records complete. Sampling successful debug traces can limit storage, but sampling the only record that connects a signup to its accepted message makes a later support investigation probabilistic. Record every retry decision in a compact form if the evidence policy requires that history; discard repeated unchanged polls rather than treating every read as a new lifecycle event.

A mail-open signal is a poor substitute for application verification. Apple's Mail Privacy Protection documentation is relevant to interpreting mail activity. The ledger should record the application's actual validation of the link, with its own timestamp, separately from any provider-reported interaction or transport outcome.

## Decide what the investigation will lose

Thirty days is an arithmetic assumption here, not a healthcare retention rule. The appropriate evidence period requires a documented purpose, applicable obligations, and an approved handling policy. Use separate retention decisions for diagnostic traces and the verification ledger; neither a queue's lifetime nor a provider's default history establishes the application's compliance policy.

Deliberately stop keeping repeated unchanged polling responses, verification tokens in telemetry, and complete payload copies at every processing boundary. Preserve the accepted message identity, meaningful state transitions, and the application's verification result under the approved evidence policy. An investigation can then reconstruct the decisions the system recorded, but cannot replay every observation or recover a discarded response body. That is the cost of the smaller retention term. If a review requires those intermediate observations, budget for them explicitly before deleting them.

## Further reading

References:

- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
- Postmark message streams: https://postmarkapp.com/message-streams
- Apple Mail Privacy Protection: https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
- Twilio guidance on preventing toll fraud: https://www.twilio.com/docs/verify/preventing-toll-fraud
- curl retry behavior and response handling: https://curl.se/docs/manpage.html
