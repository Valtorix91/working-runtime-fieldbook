# How to Evaluate Batch Audio Transcription API for Long Recordings — 4 Evidence Tiers

Short answer: for a B2B SaaS hiring system, send long interview recordings to an asynchronous transcription job, accept webhook notifications idempotently, and retain evidence in four deliberate tiers. Keep the rubric score and a small, reviewable excerpt for the long term. Keep raw audio and full transcripts only as long as the hiring policy requires. The important design choice is which evidence is necessary to defend a structured candidate score, and which bytes merely make the observability bill larger.

A transcript is an input to a hiring decision, not the decision itself. The scoring service should validate a fixed rubric, record the source segment for each criterion, and reject incomplete structured output rather than quietly turning a partial transcript into a confident number.

## What does the retention bill actually contain?

Start with a ledger, not a vendor comparison. For each recording, estimate four quantities:

| Tier | Retained artifact | Operational purpose | Default retention question |
| --- | --- | --- | --- |
| 1 | Job metadata, status, checksum, rubric version | Reconciliation and replay decisions | Can this job be identified and reconciled? |
| 2 | Structured score, criterion decisions, model version | Hiring review and aggregate analysis | Can a reviewer understand the decision path? |
| 3 | Short evidence excerpts with timestamps | Human verification of a criterion | What is the smallest useful citation? |
| 4 | Original audio and full transcript | Reprocessing, dispute review, or policy obligation | What event justifies keeping the source? |

The dominant term is usually Tier 4 because it combines large objects with long retention. A useful planning equation is `retained bytes = recordings x average bytes x retained copies x retention window`. A 62-minute recording, a 500 MB raw object, and a batch of 10,000 recordings already yields about 5 TB before replicas, transcript indexes, failed-job remnants, or diagnostic copies. I count those copies separately. A log line containing the full transcript is a second transcript, not “just debugging.”

A ledger row should carry the quantities that make that estimate auditable:

```json
{
  "job_id": "int_01842",
  "audio_seconds": 3720,
  "raw_bytes": 524288000,
  "tier_4_expires_at": "2026-11-30T00:00:00Z",
  "rubric_version": "sales-engineer-v3"
}
```

The least complex cost control is a deletion schedule tied to evidence purpose. After the review window closes, remove Tier 4 unless a documented dispute, legal hold, or reprocessing requirement exists. Preserve Tier 2 and Tier 3 only when access control and retention policy permit it. The trade-off is real: a later appeal may have less source material. That is the price of keeping less on purpose, and it should be accepted explicitly.

## How should batch transcription handle long audio recordings?

A rubric should decide the transcript shape before the transcription job starts. For example, a B2B SaaS hiring rubric might contain four criteria: requirements discovery, technical explanation, risk communication, and collaboration. Each criterion needs a bounded score, a reason, and a timestamped excerpt. “Strong communicator” is not a data contract.

The scoring request should treat missing evidence as an explicit state. A missing excerpt is different from a zero score. That distinction prevents a failed transcription, an unsupported inference, and a genuinely weak answer from collapsing into the same outcome.

A generic submission can carry an immutable rubric version and a source checksum:

```bash
curl -X POST https://transcription.example.invalid/v1/ai/batch/submit \\
  -H 'Content-Type: application/json' \\
  -d '{
    "audio_url": "https://media.example.invalid/interviews/int-01842.wav",
    "language": "en-US",
    "metadata": {
      "candidate_id": "cand_2048",
      "rubric_version": "sales-engineer-v3",
      "source_sha256": "sha256:replace-with-computed-digest"
    },
    "callback_url": "https://hiring.example.invalid/hooks/transcription"
  }'
```

The endpoint is generic. The contract matters more than the provider name: a job identifier must remain stable, the callback must be authenticated, and the application must be able to fetch final output after a notification is lost. A callback says that state may have changed; it should not be treated as the transcript payload or proof that downstream scoring succeeded.

## What should a webhook consumer do after a retry?

Assume duplicate delivery. Assume a callback arrives before a dependent object is visible. Assume the process dies after persisting the event but before advancing the workflow. These are ordinary distributed-system states.

Persist the provider job identifier and the callback event identifier under a uniqueness constraint. Verify the signature against the raw request body, then acknowledge only after the event is durably recorded. A worker can claim the recorded event and fetch the current job state. If the same event arrives again, the uniqueness constraint makes the second delivery harmless.

The retry policy should respect HTTP semantics. A request that is safe to repeat needs an idempotency design; a request that creates a new job on every retry can multiply transcription work. Keep the client request identifier separate from the provider job identifier so the system can distinguish “we submitted this once” from “the provider assigned this job.” RFC 9110 describes the semantics that make retry decisions defensible, but it does not turn an arbitrary endpoint into an idempotent one.

Here is the smallest callback-shaped request used in an integration test:

```bash
curl -X POST https://hiring.example.invalid/hooks/transcription \\
  -H 'Content-Type: application/json' \\
  -H 'X-Signature: test-signature' \\
  -d '{
    "event_id": "evt_70031",
    "job_id": "job_01842",
    "state": "completed"
  }'
```

Deliver that body twice and assert one durable event, one scoring task, and one final score. Deliver a failed state too, because a failed transcription must not be represented as a candidate who failed every rubric criterion. The scoring worker should require a complete transcript, a matching checksum, and the expected rubric version. Otherwise it records “not scorable” with a reason and stops.

One trap deserves a short sentence. Do not log the transcript to prove that the webhook worked.

## How do you sample telemetry without losing the hiring story?

Telemetry has two jobs here: find operational failure and explain a particular score. Sampling can satisfy the first while damaging the second. I separate decision evidence from diagnostic telemetry. Every score keeps stable identifiers, rubric version, input checksum, state transitions, and evidence references. High-volume transport logs may be sampled, but the ledger row for a scored interview is not sampled away.

Labels need the same discipline. A tenant identifier, job identifier, candidate identifier, and criterion name can create a high-cardinality metrics system whose cost grows with business activity. Put those values in trace attributes or a protected audit store when investigators need them. Keep metrics dimensions bounded: outcome, coarse duration bucket, failure class, and workflow stage are usually enough to answer “where are jobs stuck?”

The useful counters are boring:

- submitted jobs by coarse recording-duration bucket
- completed, failed, expired, and not-scorable jobs
- callback duplicates and signature failures
- scores missing evidence references

Do not use a dashboard as the source of truth for the score. A metric can tell you that 2.1% of jobs failed in a window; it cannot tell a reviewer why candidate `cand_2048` received 3 for risk communication. That explanation belongs in the structured result and its permitted evidence excerpt.

I initially treated full transcript retention as the safest audit strategy. The flaw was that it made every future consumer an accidental owner of sensitive interview content. A smaller evidence tier is harder to reconstruct, but it gives access review, deletion, and incident response a bounded surface. The correction is to make source retention an exception with an owner and an expiry.

## A decision rule for production readiness

Call the workflow ready when it can answer five questions for one candidate without searching raw logs:

1. Which recording and checksum produced the transcript?
2. Which rubric version produced the structured score?
3. Which criterion is supported by which timestamped excerpt?
4. Did duplicate or delayed callbacks change the result?
5. Which retained artifact can be deleted now?

Run those questions against a short recording, a long recording, a duplicate callback, a malformed structured response, an expired source URL, and a provider timeout. The long case matters because duration changes queue time, object size, retry exposure, and review latency. The malformed case matters because syntactically valid JSON can still omit the evidence needed for a defensible decision.

A production record might look like this:

```json
{
  "job_id": "int_01842",
  "status": "scored",
  "rubric_version": "sales-engineer-v3",
  "score": 14,
  "criteria": [
    {
      "name": "risk_communication",
      "score": 4,
      "evidence": [{"start_seconds": 1842, "end_seconds": 1871, "excerpt_ref": "ex_991"}]
    }
  ],
  "source_checksum": "sha256:replace-with-computed-digest",
  "raw_audio_expires_at": "2026-11-30T00:00:00Z"
}
```

That record is useful precisely because it is incomplete by design. It does not pretend to be a permanent archive of a person’s conversation. It preserves enough to inspect the decision, reconcile the asynchronous workflow, and explain why the source is no longer present. **The practical optimization is evidence selection, not blind deletion or indefinite retention.**

## Further reading

- https://www.rfc-editor.org/rfc/rfc9110
- https://www.promptingguide.ai
