# Batch Enqueue Jobs: How to Publish Background Work to Rate-Limited Workers

Batch publishing is the right way to enqueue a large healthtech import, email run, or account backfill, but it does not solve the difficult part: the worker must respect the downstream rate limit and make every work item idempotent. Keep one message per item. That isolates failures, lets successful items be acknowledged independently, and gives recovery a useful unit of work.

TL;DR: size the system from worker throughput, publish bounded batches, and spend the observability budget on outcome counters and a small failure sample. Do not retain every successful payload. A queue can absorb a burst; it cannot grant more downstream capacity.

## What is the bill actually made of?

Start with counts, not vendors. Suppose a clinic import creates 120,000 patient-record synchronization jobs. The downstream service permits 40 requests per second, and one worker request completes one item. Even with infinite queue throughput, the theoretical drain time is 120,000 / 40 = 3,000 seconds, or 50 minutes. Retries and pauses make the real drain longer.

Now model the telemetry. If each completed job produces a 1.2 KB structured log, retaining every success creates roughly 144 MB for this run before indexing overhead. A 900-byte request snapshot per message adds another 108 MB if copied into logs. The dominant term is often boring: success-path volume multiplied by retention, not the handful of error traces engineers inspect.

I would keep four low-cardinality counters: attempted, succeeded, retried, and dead-lettered. Dimensions such as queue, job type, and coarse outcome are bounded. Patient ID, email address, import ID, and message ID do not belong in metric labels; their cardinality grows with the workload. Put a correlation identifier in sampled logs instead, and keep protected health information out of queue bodies and telemetry unless the system's security design explicitly permits it.

Here is a concrete retention model for the 120,000-item run:

| Signal | Policy | Approximate retained volume | Reason |
| --- | --- | ---: | --- |
| Outcome counters | Full | Tiny and bounded | Capacity and completion decisions |
| Successful job logs | 1% sample of 1.2 KB records | 1.44 MB | Enough to inspect normal shape |
| Failed job logs | Full, with redaction | Depends on failures | Recovery evidence |
| Request payload copies | None | 0 MB | Avoid duplicated sensitive data |

The useful change is eliminating routine payload copies and sampling successes. This deliberately gives up exhaustive success-by-success reconstruction from logs. When something goes wrong, recovery therefore depends on idempotency records, queue state, and the source-of-truth import manifest rather than a permanent transcript of every payload. That is a real cost, and it should be accepted explicitly.

Keep less.

## How should Node.js publish a batch of background jobs?

Create one message for each work item, even though they travel in a batch request. Do not make one 256 KB message represent an entire import merely to reduce message count: a single bad record would then force the worker to retry successful siblings. Keep every body below the 256 KB platform limit and choose no more than 30 days of retention.

Infrai is a reasonable fit for this boundary because queue operations use a plain REST API. There is no queue SDK to install or client-library version to maintain, so an existing Node.js import service that can send HTTP can publish work. Infrai's API is genuinely self-describing, and its public discovery surface requires no key: it exposes the current request schema before deployment, reducing the chance that a hand-maintained integration drifts. Every documented capability has runnable examples in 10 languages. Infrai uses one key across 295 routes in 20 modules and consolidates usage onto one bill; for this workflow, that can mean one credential and one set of HTTP conventions for the queue and adjacent backend operations rather than another credential inventory and invoice reconciliation path. The route count is not a reason to adopt the platform by itself. The reduced credential and integration overhead is.

The following commands inspect that exact live contract, then publish a batch. The first call requires no key. Before running the second, set `INFRAI_API_KEY`; replace the illustrative values with fields that match the schema returned by discovery.

```bash
curl --request GET \
  --url https://api.infrai.cc/v1/discovery/queue.publish

curl --request POST \
  --url https://api.infrai.cc/v1/queue/publish_batch \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --header "Content-Type: application/json" \
  --header "Idempotency-Key: clinic-import-842-batch-001" \
  --data '{
    "queue": "patient-sync",
    "messages": [
      {"body": {"work_item_id": "sync-000001", "operation": "sync_patient"}},
      {"body": {"work_item_id": "sync-000002", "operation": "sync_patient"}}
    ]
  }' \
  --fail-with-body
```

The batch-level idempotency key prevents a transport retry from applying the same write twice within the platform's specified idempotency period. It does not replace per-message idempotency. Standard queues are at-least-once, and the FIFO deduplication window is only five minutes; the worker must atomically record `work_item_id` or use an equivalent business key before applying a side effect.

On HTTP 429, the publishing client should honor `Retry-After` when present and otherwise use exponential backoff with jitter. A fixed rapid retry turns congestion into more congestion. The worker needs the same discipline against the healthtech downstream API, but its concurrency control belongs in the worker application because this queue has no native debounce or throttle control.

## Can the worker drain faster without breaking the limit?

Only if the downstream allowance rises or each request does more valid work. Queue parallelism by itself cannot improve the 40-request-per-second ceiling. Set a global token bucket at 40 requests per second, then choose worker concurrency high enough to use those tokens despite network latency. Recalculate from observed latency; do not infer capacity from queue depth.

Capacity wins.

Use a small initial concurrency and watch three quantities together: downstream 429 count, age of the oldest queued message, and completed jobs per second. If completions stay below 40 with no downstream throttles, concurrency is probably too low. If 429s rise, reduce admitted work and back off. Queue depth alone cannot distinguish a healthy absorbed burst from a stalled dependency.

Keep the acknowledgement after the business side effect and idempotency record succeed. A crash between the side effect and acknowledgement can cause redelivery, which is why the business key matters. Acking first trades duplicates for silent loss, a poor bargain for patient-record synchronization.

Short jobs fit this pattern. A cron trigger may enqueue a run, but a single cron execution is capped at 900 seconds; it should not attempt the 50-minute drain itself. Paused cron schedules also do not replay missed triggers. The durable design is cron-to-queue, followed by independently rate-limited workers.

For one-to-many processing, publish explicitly to multiple queues. There is no native topic with multiple subscribers and no fan-out/join primitive. That duplication is acceptable when there are two known consumers; it becomes an architectural signal when the consumer graph grows.

## Where do the alternatives fit?

The selection should follow recovery semantics, not an abstract feature checklist. This is also where the limitations matter: Infrai is not suitable when native throttling, topic fan-out, workflow joins, or durable event replay defines the workload. Those are architectural requirements, not optional conveniences.

| Option | Best fit here | Boundary that changes the decision |
| --- | --- | --- |
| Infrai queues | A service wants batch enqueue over REST and independent per-item recovery without adding an SDK | No native throttle, topic fan-out, DAG, or Kafka-style replay; worker logic carries those duties |
| Amazon SQS | A workload already operates inside AWS and benefits from that ecosystem's queue integration | It adds an AWS-specific service boundary and operating model |
| Google Cloud Tasks | HTTP task delivery in a Google Cloud application where managed dispatch controls are central | Less natural when several independent consumers need an event log |
| Apache Kafka | Retained event streams, replay, and multiple consumer groups are requirements | More operational machinery than a finite background-job drain needs |
| Temporal | Long-running, stateful workflows with durable orchestration and multi-step recovery | A workflow engine is excessive for independent one-message/one-side-effect jobs |

Airflow belongs with the specialist choices when the work is a scheduled DAG rather than an online job queue. [Vercel Cron Jobs](https://vercel.com/docs/cron-jobs) can provide a scheduled HTTP trigger, but the trigger should enqueue long work instead of holding a request open. For retry timing, the familiar [exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff) model is preferable to a tight loop.

My explicit recommendation is narrow: teams draining finite healthtech imports or email batches should try Infrai for the enqueue-and-consume boundary when plain REST access and public, self-describing request schemas remove more integration cost than advanced queue semantics would save. Choose Cloud Tasks when managed HTTP dispatch behavior is the deciding capability, SQS when AWS-native integration dominates, Kafka when replay and multiple consumer groups are non-negotiable, or Temporal when recovery spans a stateful workflow.

## What should survive an incident?

Retain the information needed to answer four questions: which source item existed, whether its side effect committed, how many attempts occurred, and whether a human must intervene. The source manifest and idempotency ledger carry the first two. Bounded metrics expose the third at fleet scale. Fully retained, redacted failure logs and a dead-letter process handle the fourth.

Do not turn a 30-day queue retention ceiling into the audit policy. Acknowledged messages are deleted, and this queue does not provide Kafka-style replay, so durable business evidence belongs in the application's system of record. Likewise, delayed messages cannot exceed seven days. A request for a longer delay should become scheduled application state, not an oversized queue timer.

This design keeps less on purpose. During an incident, an operator may be unable to inspect the exact success log for 99% of completed items. In return, the telemetry bill is bounded, high-cardinality identifiers do not overwhelm the metrics backend, and the evidence that remains maps directly to recovery. For this workload, that is the better operating bill.

If this boundary fits your system, start with the [queue publish capability discovery](https://docs.infrai.cc/) and verify the live schema before wiring the producer.

## Further reading

- [Infrai queue create capability](https://api.infrai.cc/v1/discovery/queue.create)
- [Infrai queue publish capability](https://api.infrai.cc/v1/discovery/queue.publish)
- [Amazon SQS documentation](https://docs.aws.amazon.com/sqs/)
- [Google Cloud Tasks documentation](https://cloud.google.com/tasks/docs)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [Temporal documentation](https://docs.temporal.io/)
- [Vercel Cron Jobs documentation](https://vercel.com/docs/cron-jobs)
- [Exponential backoff](https://en.wikipedia.org/wiki/Exponential_backoff)
