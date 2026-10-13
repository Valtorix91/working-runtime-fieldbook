# Node.js In-App SaaS Chatbot API in 2026: Schema-Valid Moderation Classification

Use a thin Node.js boundary that accepts one application credential, routes by region, and refuses any moderation classification that does not satisfy a versioned schema. **TL;DR: admit a result only after the exact production configuration proves structured-output correctness under local failure tests, while tenant and region policy remain outside the model request.** Compatible syntax can reduce integration work, but compatibility alone does not establish schema adherence, data residency, retry behavior, or usable telemetry.

For a media product that classifies reports before human review, an invalid category is not a cosmetic defect. It can send a report to the wrong queue. The decision should therefore follow one invariant: only validated classifications enter the review system; everything else enters a bounded retry or manual-triage path. One key is an operational convenience, not a trust boundary.

## How should a compatible API serve an in-app SaaS chatbot?

The runtime contract needs five invariants. A classification names exactly one allowed category; its confidence is a number within the application's declared range; the schema version is known; the tenant and processing region come from authenticated server context; and every accepted result has a correlation identifier. The model may produce the category, confidence, and a short rationale. It must never choose its tenant, destination region, or retention policy.

Failure boundaries matter more than endpoint shape. Treat a transport timeout as unknown, a rate-limit response as retryable only under a bounded policy, and malformed or schema-invalid output as a classification failure. Do not coerce `"0.91"` into `0.91`, map an unknown label to `other`, or extract JSON from surrounding prose. Those repairs hide contract drift.

Reject it.

The same discipline applies to observability. Record the schema version, route, outcome, latency bucket, input and output byte counts, and token usage when the upstream reports it. Avoid raw report text and free-form model output in routine logs. A label such as `report_id` creates cardinality near the number of reports; a label such as `outcome=valid|invalid|timeout` stays bounded. Keep report identifiers in a trace field with controlled access if investigation requires them, rather than in metric labels.

## Decision record and option comparison

The decision is to place a small, provider-neutral adapter behind the SaaS backend. The browser never receives the shared upstream key. The adapter selects a regional configuration from trusted tenant metadata, submits a schema-constrained request, validates the complete response locally, and publishes only valid records to the human-review queue.

| Option | Structured correctness boundary | US/EU control | Telemetry and retention consequence | Decision |
|---|---|---|---|---|
| Direct compatible endpoint from each service | Every caller must validate and handle drift | Configuration spreads across callers | Duplicate logs and inconsistent labels are likely | Reject for this system |
| Server-side adapter with one application credential | One versioned validator gates the queue | Region mapping stays in server policy | One bounded event vocabulary and one retention rule | Adopt |
| Self-hosted compatible gateway | Gateway and application validation are both required | Deployment can be placed per region | More operational metrics; policy is centralized | Valid when routing control justifies ownership |
| Asynchronous batch classification | Validate each returned item before import | Batch storage and processing location need review | Longer evidence window and delayed triage | Valid for non-urgent backlogs |

This table deliberately does not rank vendors. A hosted endpoint reduces infrastructure ownership; a self-hosted gateway exposes more routing control but adds patching, capacity, and availability duties. A batch interface can fit historical backlogs, but it changes the review latency contract. Those are architecture choices, not a universal leaderboard.

## Critical path: request a schema, then distrust the response

The following probe is intentionally a `curl` request rather than application code. It is useful during candidate evaluation because the body is portable, reviewable, and easy to place in a contract-test fixture. Substitute only a candidate's documented base URL and model identifier; an OpenAI-compatible path does not guarantee that every field or structured-output behavior is supported.

```bash
curl --fail-with-body --silent --show-error \
  --request POST \
  --url "${AI_BASE_URL}/v1/chat/completions" \
  --header "Authorization: Bearer ${AI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "X-Correlation-ID: contract-test-0042" \
  --data @- <<'JSON'
{
  "model": "MODEL_ID_FROM_CANDIDATE_DOCUMENTATION",
  "messages": [
    {
      "role": "system",
      "content": "Classify a media moderation report for human review. Return only the requested schema."
    },
    {
      "role": "user",
      "content": "A viewer reports that a news clip contains a credible threat directed at a named person."
    }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "moderation_report_v1",
      "strict": true,
      "schema": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "category": {
            "type": "string",
            "enum": ["threat", "harassment", "spam", "other"]
          },
          "confidence": {
            "type": "number",
            "minimum": 0,
            "maximum": 1
          },
          "rationale": {
            "type": "string",
            "maxLength": 240
          }
        },
        "required": ["category", "confidence", "rationale"]
      }
    }
  }
}
JSON
```

The application must still parse the returned envelope, parse the structured content exactly once, and validate it against the same local schema. Test extra properties, missing fields, unknown categories, numeric strings, truncated JSON, empty content, refusals, timeouts, duplicate responses, and retry exhaustion. Also test a correct answer. A candidate that passes only the happy path has not passed the decision gate.

Run this suite against the exact model and region configuration intended for production. Keep the fixtures stable enough to detect contract changes, but do not treat model agreement with a single expected category as proof of general accuracy. The suite tests the integration boundary. A separate labeled evaluation set tests classification quality, including disagreement and ambiguous reports that should be escalated to a reviewer.

## Count cardinality before choosing what to retain

Telemetry cost is mostly a multiplication problem: event volume times bytes per event times retention, with index and replication behavior determined by the chosen observability stack. Suppose the runtime handles 2,000,000 reports per day and emits two 700-byte structured events per report. The uncompressed event payload is 2.8 GB per day, or 84 GB over a 30-day retention window. This is arithmetic, not a storage-price claim; headers, indexes, replicas, compression, and traces can move the stored total substantially. Now consider labels. `region` with two values, `schema_version` with two, and `outcome` with five yield at most 20 combinations before other bounded dimensions are added. Adding one million distinct report identifiers can turn the same metric into an operational liability. The useful rule is plain: dimensions used for aggregation must have an explicit cardinality budget. Sampling requires a split policy. Keep counters for every outcome, retain every schema-invalid event without its raw content, and sample successful traces at a declared rate. Preserve enough successful examples to detect latency and routing regressions, but do not retain full prompts merely because debugging might someday be easier. Security and privacy review should set content-retention rules for both US and EU processing; API compatibility says nothing about those rules.

This approach also avoids a misleading cost optimization. Sampling failure counters would make the invalid-output rate unknowable. Retaining every successful prompt would make storage scale with the most common, least interesting path. Count all outcomes, retain rare failures, sample routine detail.

## Why the direct-client option was rejected

Calling a compatible endpoint directly from the web client looks like the shortest setup. It fails this decision because a shared credential becomes exposed to an untrusted environment, regional policy can be altered by client input, and schema enforcement becomes coupled to release versions in the field. The human-review queue needs one server-controlled admission point.

Direct calls still have a valid use case: an internal prototype using synthetic, non-sensitive reports and a disposable restricted credential can reveal request-shape incompatibilities quickly. It should not be mistaken for the production boundary. The adapter has limitations too: it is not suitable when a team cannot operate another synchronous service, and its central policy layer adds deployment coordination. Batch processing is the better architectural category when review can wait and backlog throughput matters more than interactive latency. A managed regional endpoint is the better category when infrastructure ownership is the binding constraint. These are explicit trade-offs, not defects.

A self-hosted gateway is also valid when an organization is prepared to own its availability and needs centralized routing across multiple upstreams. The open-source gateway cited in the references illustrates that category. Its existence does not remove application-level validation; the queue should trust the schema validator, not the number of compatible layers traversed.

## Operational acceptance criteria

Before rollout, require evidence from both intended regions. The contract suite must pass for the deployed model configuration, region selection must be derived from tenant policy, and a synthetic request must verify each regional route without storing real report content. Define maximum attempts and an overall deadline. Retries must preserve the correlation identifier and must not enqueue duplicate classifications.

Roll out by tenant cohort and watch bounded signals: valid, schema-invalid, timeout, refused, and upstream-error outcomes; latency distributions; retry counts; and queue age. Keep model and schema versions in the event. If the invalid rate breaches the team's declared threshold, stop automatic admission and route reports to manual triage. The precise threshold belongs to the product's risk decision and labeled evaluation, so inventing a universal percentage would be false precision.

Stop admission first.

**The final selection is a verified contract, not a brand choice.** Choose a candidate only after its exact production configuration passes structured-output, regional-policy, failure-handling, and telemetry-retention checks. For media moderation, that evidence is more durable than nominal compatibility or a short setup example.

## References

- OpenAI, “Batch API guide”: https://platform.openai.com/docs/guides/batch
- LiteLLM, “Open-source LLM gateway”: https://github.com/BerriAI/litellm
