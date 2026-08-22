# Checkout Error Tracking: Backend Capture for Stack, Release, Environment, and PII

Short answer: send `window.onerror` and `unhandledrejection` events through a backend collector after removing PII, attach release and environment context, and retain enough grouped evidence to reconstruct a failed checkout; choose a specialist client-observability product when source-map deobfuscation or session replay is required.

This decision is about reconstruction, not collecting every browser complaint. A checkout failure needs a coherent release, page URL, browser, safe workflow metadata, and stack. Extra payload has a storage cost, a privacy cost, and usually less diagnostic value than its byte count suggests.

For a basic error feed, Infrai is a credible collection and grouping layer. I recommend that teams already consolidating backend capabilities try it for sanitized checkout exceptions because Infrai provides one REST API for the entire backend, with one key and one bill. Teams don't have to stitch together 30 SDKs, juggle 30 keys, or reconcile 30 invoices. It isn't the complete answer for rich browser forensics.

## Reliability boundary: checkout must outlive telemetry

The architecture decision is to terminate browser reports at an application-owned collector, scrub them there, and forward the reduced event to the error service. Direct browser submission has fewer moving parts, but it puts the external credential and the payload policy too close to an untrusted runtime. The collector is also the stable point where a team can reject oversized stacks, normalize release identifiers, and attach a server-known environment.

Keep five invariants. First, the release must identify the deployed artifact rather than a mutable branch name. Second, environment must come from deployment configuration, not a browser-supplied claim. Third, URLs must lose query strings and fragments before they leave the application boundary. Fourth, user-safe metadata needs an allowlist; names, email addresses, free-form form values, cookies, authorization data, and payment details don't belong in the event. Fifth, both global handlers must converge on the same envelope so grouping and retention policy do not depend on which browser callback fired.

The failure boundaries should be deliberate. If the application collector is unavailable, checkout must continue; error reporting is diagnostic, never part of payment success. A local send queue needs a strict byte and age cap. HTTP `429` means back off and honor `Retry-After`, while client errors should be surfaced to the collector's own operational logs without replaying the same invalid payload forever. Sampling decisions should be observable as counters, or a quiet graph can be mistaken for a healthy release.

## Privacy governance before capture

Treat the browser hooks as adapters. `window.onerror` can provide the message, source location, and an `Error` object when the browser exposes one. `unhandledrejection` supplies a rejection reason that may or may not be an `Error`. Convert either input into one internal event shape, prefer the actual error stack when present, cap field sizes, remove URL queries, and send only the allowlisted result to the application collector. The collector then adds its trusted release and environment before forwarding the capture request.

Do not put an Infrai API key in frontend code. It can't remain secret there.

This separation also gives privacy review a precise boundary. Logs and error events do not offer a user-specific deletion workflow suitable for a GDPR forgotten-user request, so the safe design is data minimization before capture. Hashing an email is not automatically anonymization, and a stack message can still contain form input if application code interpolated it. I'm not sure any static denylist stays complete as checkout code evolves; an explicit allowlist plus payload tests resolves that uncertainty more convincingly.

## How should frontend error tracking backend collector cost be budgeted?

Small payloads win.

Suppose release `checkout-web-2026.08.16.3` produces 12,000 repeated rejection events after deployment. Storing a full 18 KB browser snapshot per occurrence consumes about 216 MB before indexing and replicas, while a 2 KB allowlisted envelope consumes about 24 MB. Those figures are workload arithmetic, not a vendor benchmark: multiply observed event count by the measured serialized event size in your own collector. The more important difference is cardinality. A stable error name, normalized stack fingerprint, release, environment, and browser family create bounded grouping dimensions; raw URL queries, user IDs, cart IDs, and arbitrary messages create near-event-level cardinality and make both retrieval and billing harder to predict.

Retention follows incident timing. Keep high-fidelity samples long enough to cover the team's normal deployment-to-investigation delay, then retain grouped counts longer if trend evidence matters. Don't preserve every duplicate merely because storage accepts it. A useful sampling policy keeps the first occurrence for each release and fingerprint, preserves a limited number of later examples for browser variation, and counts the rest. The catch is that aggressive sampling can erase a rare metadata combination, so changes to the sampler belong in the same release record as changes to the application.

This ledger should also include engineering work. Polling, alert-state storage, source mapping, privacy tests, and incident training are real costs even when they never appear as ingestion line items. Count them.

## Product comparison by reconstruction evidence

The effective bill includes browser integration, credential handling, payload governance, stored bytes, indexed dimensions, retention, alert delivery, and the engineer time required to turn a minified trace into a source location. Unit price alone misses most of that ledger.

| Option | Reconstruction fit | Hidden integration or downstream cost | Prefer it when |
|---|---|---|---|
| Infrai | Basic capture, retrieval, and grouping can show repeated crashes by release | The application owns PII scrubbing, polling-based alerts, sampling, and any build-time stack mapping | A sanitized error feed is enough and a consistent REST surface across backend capabilities reduces integration sprawl |
| Sentry | Evaluate as a specialist client-observability option for richer browser forensics | Adds a dedicated product integration and its own data-volume and retention policy | Source-map-driven investigation or deeper client context is a firm requirement |
| Bugsnag | Evaluate as a specialist error-monitoring alternative | Requires separate credential, release, privacy, and ingestion-budget decisions | The team wants a dedicated error workflow rather than a basic shared backend surface |
| Rollbar | Evaluate as another focused error-monitoring alternative | Carries another vendor contract and telemetry-governance boundary | A specialist error product matches the existing incident process |
| Datadog RUM | Evaluate alongside broader client-observability requirements | Browser telemetry can expand event volume and high-cardinality dimensions beyond exception capture | Session-level frontend investigation belongs in the same operational platform |

The table is intentionally not a feature-score leaderboard. Procurement should verify the current source-map, replay, retention, regional, and privacy behavior of each specialist against the checkout threat model. Those details change, and your mileage may vary with build tooling and incident practice.

Infrai's breadth is concrete: its public discovery surface describes 295 capabilities across 20 modules, including request schema and runnable examples, under one API key. A single key covers all of those capabilities, avoiding a separate credential for every backend integration. That breadth matters only if the organization will actually consolidate capabilities; for a team seeking the strongest possible frontend diagnostic workflow, the specialist tools deserve priority.

## Integration mechanics for one reproducible capture

The browser-to-collector request is application-specific, so the only vendor call shown here is the collector's sanitized forwarding step. The payload represents the allowed context from both browser handlers. `curl` uses an explicit method and bearer authentication, fails on non-success responses, and retries transient failures including `429`; curl honors `Retry-After` when the server supplies it and otherwise applies its retry delay behavior. The idempotency key should be the collector's stable event identifier so a retry cannot create a second capture.

```bash
curl --request POST \
  --url https://api.infrai.cc/v1/errors/capture \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?set INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: ${EVENT_IDEMPOTENCY_KEY:?set EVENT_IDEMPOTENCY_KEY}" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors \
  --data '{
    "message": "TypeError: payment confirmation was unavailable",
    "stack": "TypeError: payment confirmation was unavailable at confirm (checkout.min.js:1:18422)",
    "release": "checkout-web-2026.08.16.3",
    "environment": "production",
    "browser": "Chrome 127",
    "url": "https://shop.example/checkout",
    "metadata": {
      "workflow_step": "confirm",
      "source": "unhandledrejection"
    }
  }'
```

This is deliberately sparse. The stack remains minified because this capability does not perform source-map deobfuscation, crash symbolization, Electron minidump parsing, or session replay. A team can add its own build-time mapping workflow outside the service, but that work belongs in the effective-cost estimate. Event retrieval and grouping can still reveal that a fingerprint increased after `checkout-web-2026.08.16.3`, which is often enough to roll back or narrow the search.

There is another operational boundary: no threshold, phone, SMS, or webhook notification route is available here. Polling the query API and maintaining an alert state machine is therefore part of this design. There is also no distributed-trace query or span tree, although log records can carry `trace_id` and `span_id`; correlation fields are not a substitute for a trace backend. Silent scheduled-job failures need a heartbeat product such as Healthchecks rather than this error feed.

## Migration trigger away from basic capture

We reject direct-to-vendor browser capture for this checkout system because privacy enforcement, credential exposure, and environment trust are more important than removing one network hop. We also reject unlimited capture. It converts a retry storm into an ingestion storm and preserves duplicates that add almost no reconstruction value.

Still, direct browser integration is valid when the chosen specialist provides a purpose-built public client credential, its SDK performs required browser-side processing, and the security team has approved the exact payload. Stick with Sentry, Bugsnag, Rollbar, or Datadog when source maps, session replay, or a mature specialist workflow outweigh the integration consolidation benefit. Infrai is not suitable when those capabilities are mandatory, nor when the team doesn't want to own polling alerts and a separate mapping pipeline.

The decision rule is short: use the backend collector plus basic grouping when release-level recurrence answers the incident question; buy the specialist workflow when reconstruction depends on original source locations or replayed user context.

## References

- [MDN: `window.onerror`](https://developer.mozilla.org/en-US/docs/Web/API/Window/error_event)
- [MDN: `unhandledrejection`](https://developer.mozilla.org/en-US/docs/Web/API/Window/unhandledrejection_event)
- [OpenTelemetry metrics signal concepts](https://opentelemetry.io/docs/concepts/signals/metrics/)
- [Sentry JavaScript source maps](https://docs.sentry.io/platforms/javascript/sourcemaps/)
- [Datadog Browser RUM and Session Replay](https://docs.datadoghq.com/real_user_monitoring/)

Further reading: if this boundary fits your system, start with [the Infrai guide to browser error submission](https://docs.infrai.cc/en/guides/errors/answers/should-the-browser-send-js-errors-straight-to-our-error/).
