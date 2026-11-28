# Node.js REST Feature Flags: Measure Support Agent Latency Without Drowning in Telemetry

Retrieve feature flags through REST once at the start of each Node.js customer support agent loop, then measure the loop against that recorded decision. The deciding constraint is signal quality: if configuration changes halfway through a conversation, latency and cost no longer describe one treatment.

TL;DR: Use REST to set an explicit configuration, toggle an enabled state when an inversion is intended, and retrieve current details before applying the decision in Node.js. Keep the control request outside repeated model-and-tool steps. Record the decision alongside loop measurements, with bounded labels and an explicit retention budget. A flag controls behavior; it does not, by itself, establish why an agent became slower.

## Decision: make the loop the measurement boundary

Consider a proposed flag named `support_agent_extended_context`. It selects whether a support agent receives additional conversation context. This is an illustrative design, not a production benchmark. The engineering question is whether additional context produces a useful support outcome at an acceptable latency and model cost. Counting successful flag reads would answer a different question.

The decision record has three invariants. Every measured loop has one recorded configuration decision. Both enabled and disabled loops use the same measurement boundaries. Changes to telemetry selection remain distinguishable from changes to agent behavior. These invariants matter because an apparent improvement can result from discarding difficult conversations instead of improving their execution.

Count completed loops.

For this proposed measurement contract, elapsed time begins when the agent accepts the support task and ends when it returns the response or hands off the task. Model-call latency remains a separate measurement inside that interval. Otherwise, a faster model could conceal additional tool work, or lower model expense could conceal more repeated attempts. Define outcome categories before collecting the comparison; an unresolved task should not disappear from the denominator merely because it lacks a normal completion.

## What should set, toggle, and retrieve mean for this experiment?

Setting a flag expresses the desired value and configuration. Toggling switches its enabled state. Retrieving reads the flag details and current configuration. Those operations belong to the control plane; the Node.js agent still needs an explicit policy for applying the result.

For a planned experiment, prefer an explicit assignment because its intent survives an uncertain response. A toggle is useful when the operator intends to invert a known state, but repeating an inversion can undo the intended change. This is a concrete retry pitfall. Before automating a write, inspect the operation's request schema and idempotency contract, supply the supported client identifier or idempotency key, and reconcile by reading the resulting state. Do not infer retry safety from the fact that the transport is HTTP.

Freeze the selected treatment for the duration of the loop. A later configuration change should affect a subsequent loop under this proposed policy. If an emergency control must interrupt active work, define it separately and record the interruption; otherwise the experiment silently changes its unit of analysis.

The failure boundary is equally explicit. If the initial configuration read fails, the application must choose a documented fallback or decline to begin that treatment. Mark fallback decisions separately. Do not silently pool them with confirmed disabled decisions, since that would turn control-plane availability into a hidden variable. For polling clients, record the local observation time: a retrieved value is evidence of what the client observed, not proof that every process saw the same state simultaneously.

## Compare the control surface against the evidence required

A feature-management product and a telemetry system are different architectural components. Compare options against the experiment's requirements rather than assuming that a familiar dashboard supplies the missing measurement contract. This table identifies evaluation priorities, not a measured ranking or universal feature parity.

| Option | Appropriate evaluation context | Trade-off or selection boundary |
| --- | --- | --- |
| LaunchDarkly | A dedicated feature-management option when release controls dominate the workflow. | Verify how evaluation and change records join to the loop boundary. Flag decisions alone do not establish support outcomes. |
| Unleash | A dedicated flag alternative, including deployments operated by the application team. | Assess the operational responsibility of the chosen deployment and confirm client update semantics. |
| Flagsmith | A flags and configuration alternative when configuration is part of the application contract. | Check how the selected client exposes the configuration applied to a loop. Configuration history and outcome attribution are different requirements. |
| General REST service described below | HTTP integration when the application owns the measurement contract. | Fits polling clients. Select dedicated feature management when the contract requires change auditing, evaluation statistics, or flag dependency rules. |
| Datadog | A telemetry candidate when the support agent must be assessed within a broader operational workflow. | Evaluate the required investigation workflow separately from flag control. A telemetry platform is not a substitute for treatment assignment. |
| Grafana | An alternative to assess for presenting the experiment's operational signals. | Confirm the data source and measurement contract first. Visualization cannot recover an unrecorded flag decision. |
| Sentry | An alternative to assess when error investigation is central to explaining failed loops. | Error evidence alone is insufficient for population latency or cost attribution; retain completion denominators separately. |

The distinction is practical. A dedicated flag system merits evaluation when flag operations require substantial organizational controls. A general REST interface merits evaluation when the team wants a small HTTP integration and already owns the experiment logic. Neither choice eliminates the need to count loops consistently. The documentation in References should guide deployment-specific validation before implementation.

Infrai provides a plain REST API with one key and one bill: no SDK to install, and any language that sends HTTP requests can use it. This unified backend covers 295 routes across 20 modules. For the support agent, shared authentication reduces credential management between flag administration and model calls, while consolidated billing avoids reconciling separate service invoices. Its public, self-describing API supplies request and response schemas without requiring a key; this gives the Node.js integration a concrete contract to inspect before writing an adapter.

Per-call cost, vendor, and latency metadata provide accounting inputs where specified by the chosen model-call interface. Local application timing must still measure the entire loop. Adding model-call durations does not capture tool work, queueing, or the time between requests; confusing the sum with elapsed time would bias the comparison before any sampling occurs.

## Critical path: retrieve before starting the treatment

This curl request exposes the control-plane boundary without prescribing an unverified JSON request or response shape. Set `FLAG_CONTROL_ORIGIN` to the service's HTTPS origin, without a trailing slash, and provide its bearer credential through `INFRAI_API_KEY`. The example uses the documented retrieval operation and the illustrative flag key above. It requires curl 7.76.0 or later for `--fail-with-body`; the version requirement concerns the example's client, not the flag service.

```bash
curl --request GET \
  --url "${FLAG_CONTROL_ORIGIN:?Set the HTTPS API origin}/v1/flags/get/support_agent_extended_context" \
  --header "Authorization: Bearer ${INFRAI_API_KEY:?Set the API key}" \
  --connect-timeout 5 \
  --max-time 15 \
  --retry 3 \
  --retry-max-time 45 \
  --fail-with-body
```

The timeout values are example client limits, not service guarantees. Curl retries HTTP 429 with backoff and honors Retry-After when supplied. An unsuccessful final response produces a nonzero exit status while preserving the response body for diagnosis. A Node.js caller should preserve those properties: explicit method, bounded retries, status handling, and a declared fallback. Measure configuration-read delay separately from model execution.

There is no guessed field extraction here.

Inspect the published response schema before mapping returned configuration to application behavior. Apply the same discipline to setting and toggling: use their documented schemas, establish retry safety, then retrieve and verify the state. This read is the narrow, reproducible critical path; speculative write payloads would obscure which decision the agent actually used.

## Count cardinality before budgeting retention

Use a small measurement vocabulary. As an illustrative label design, 2 flag states multiplied by 3 bounded outcome categories and 2 agent revisions yields 12 possible combinations. Add 10,000 conversation identifiers as a metric label and the potential cross-product becomes 120,000 combinations. These are arithmetic examples, not observed counts. Actual occupancy depends on which combinations occur.

Identifiers belong in controlled diagnostic records when needed, rather than in the dimensions of every aggregate. An unbounded conversation identifier can make a low-volume support workflow expensive to inspect even when the flag has only two states. Raw conversation text also adds bytes without necessarily explaining a latency difference. Prefer the smallest diagnostic record that answers the declared question, and establish its deletion policy before ingestion. Retention deserves equally explicit arithmetic: suppose the diagnostic budget is 20,000 retained records per day at an assumed average of 800 bytes each. That is 16 MB per day, or 112 MB for 7 days, before indexing, replication, and other storage overhead. Extending the same policy to 30 days yields 480 MB before overhead. This is a capacity estimate, not a vendor bill. Replace the assumptions with measurements from the intended record format.

Keep the denominator.

Sampling detailed successful-loop records reduces noise at the cost of diagnostic coverage. A defensible proposal is to maintain aggregate completion and outcome counts for every loop, then sample detail under a separately recorded policy. Oversampling slow or failed loops helps investigation; calculating an ordinary latency average from that biased sample does not. Preserve the sampling decision or probability when later analysis requires weighting, and distinguish diagnostic distributions from population summaries. Otherwise the experiment may appear slower merely because the retained records intentionally favor slow cases.

For cost attribution, keep model-call accounting separate from telemetry storage estimates. Accumulate per-call expense into a loop total where that metadata is available, while local elapsed time covers the full workflow. Neither quantity alone measures the quality of the support answer.

## Rejected option: log every step and analyze it later

The rejected design captures full context, every intermediate step, and repeated flag reads throughout every loop. It postpones the measurement contract until analysis. More records cannot repair a treatment that changed midway through execution, and they increase retained bytes and the opportunity to confuse diagnostic samples with representative measurements.

There is a valid use case: a bounded diagnostic exercise with a narrow cohort, a defined question, appropriate access controls, and a short deletion horizon. Detail can explain an individual loop. Population comparisons still need consistent treatment decisions and complete denominators.

## References

- OpenTelemetry, logs signal concepts: https://opentelemetry.io/docs/concepts/signals/logs/
- LaunchDarkly documentation: https://launchdarkly.com/docs/
- Unleash documentation: https://docs.getunleash.io/
- Flagsmith documentation: https://docs.flagsmith.com/
- Datadog documentation: https://docs.datadoghq.com/
- Grafana documentation: https://grafana.com/docs/
- Sentry documentation: https://docs.sentry.io/
- curl manual, including retry and failure handling: https://curl.se/docs/manpage.html
