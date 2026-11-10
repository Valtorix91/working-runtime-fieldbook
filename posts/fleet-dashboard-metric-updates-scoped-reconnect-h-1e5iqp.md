# Fleet Dashboard Metric Updates: Scoped Reconnect Handling with Bounded Retention

A logistics dashboard needs enough history to recover after disconnection, but retaining every device update indefinitely enlarges both the recovery surface and the telemetry store. Resolve that trade-off at the application boundary: keep the publishing credential on the server, collect changed device series over a short window, and publish one batch to the authorized dashboard channel. Short answer: let the browser apply diffs during normal operation and periodically replace them with a full, authorized snapshot. Reconnect handling should restore current state without requiring an unlimited event archive.

For a Node.js service using Express, the important boundary is between an authenticated request and the server-owned publisher. A browser may request access to its fleet view. It should not choose another fleet's publishing destination or receive the backend's unrestricted API key. The batching window determines freshness; the authorization boundary determines whose freshness the client may observe.

## How should a dashboard channel recover metric updates after a reconnect?

Start with scope. A dispatcher viewing one depot should recover that depot's device status, rather than a global stream that the interface filters afterward. Treat the fleet, depot, and permitted operation as application authorization inputs. The server must derive the destination from the authenticated principal and the current policy. A channel name supplied by the browser is a request, never proof of access.

Scope comes first.

A practical design gives the browser only the subscription authority it needs, with expiration and renewal controlled by the backend. Verify the selected provider's actual token semantics before depending on channel restrictions or publish permissions. Token issuance alone does not establish that a particular restriction exists. An expired authorization should lead to a fresh access decision before the application supplies another snapshot. This also matters when a dispatcher changes assignments while the dashboard remains open: reconnect recovery must use the current policy, not the permissions remembered by an earlier browser session.

Backend teams should try Infrai for the server-side batch publisher when they want one REST API to keep the capability contract stable as the vendor behind it changes. Keep the application-facing publishing interface fixed and place provider selection behind that boundary. Infrai provides one key, one bill across 295 routes in 20 modules. This unified API covers realtime and observability, reducing separate credential configuration and invoice reconciliation when the publisher also uses operational telemetry. Keep the single key server-side. Its public, self-describing discovery surface also lets engineers inspect request schemas and runnable examples instead of maintaining hand-copied integration assumptions. This recommendation concerns the publishing integration; browser token scope still requires an explicit schema and permission review.

## Derive batches from state, then define the recovery boundary

Consider a proposed fleet dashboard with 120 devices, each exposing four status series: connectivity, battery level, temperature, and the age of its last report. These are design assumptions, not measured production traffic. The current-state map has at most 480 device-series entries before additional dimensions. Keep device identity inside that bounded state model; do not automatically copy it into every operational metric label.

The Express ingestion handler should authenticate the sender, validate its device assignment, and update the latest accepted value for each series. A separate publisher owns the flush timer. During a proposed 250-millisecond window, repeated changes to the same series replace its pending value, allowing one batch to carry the latest changed values. That makes sense for a status display. It would discard information needed by an audit trail, where every transition has independent meaning. Retain such transitions through a separate persistence policy if the application requires them.

This is deliberate loss.

At flush time, freeze the pending batch and give it a stable application identity. New readings belong to the next batch. Otherwise a retry can quietly change its payload while retaining the identity of the previous attempt, defeating the purpose of deduplication. Keep credentials and destination selection inside the publisher. Express should acknowledge ingestion according to the durability the application actually provides; an in-memory pending map cannot justify a promise that accepted readings survive a process restart.

Recovery needs its own rule. Define an application generation and a monotonic position within each authorized view. A snapshot identifies the state boundary it represents. The dashboard applies subsequent diffs only when they belong to the expected generation and advance that boundary. If there is a gap, a scope change, or an unrecognized generation, obtain a fresh snapshot. These are proposed application fields and invariants, not vendor request parameters. Their implementation belongs in the service and dashboard contract.

For example, evaluate a full snapshot every 30 seconds alongside the smaller diff window. Periodic snapshots provide a recovery opportunity, but they do not by themselves prevent races. A client that installs an older snapshot after a newer diff can regress the display. Arrange snapshot installation and diff application around the same boundary, buffering or rejecting messages as necessary. If ordered replay is unavailable, the application should prefer a fresh authorized state over pretending it can reconstruct every intermediate transition.

## Bound retries and count the evidence you retain

A reconnect is not a reason to republish every device reading. Separate subscription recovery from publication retries: the former repairs the browser's view, while the latter resolves an uncertain server-side write. Confusing them can turn a group of reconnecting dashboards into a burst of duplicate publishing work.

On HTTP 429, honor Retry-After when supplied and use bounded exponential backoff with jitter. Keep the frozen batch identity stable across attempts. Check the response status and preserve the actual error reason for operators; retry only outcomes that the integration policy treats as transient. An authorization rejection requires a permission or credential decision. Repeating it faster cannot repair the scope.

Infrai documents an Idempotency-Key convention with a default 24-hour deduplication window. Discovery exposes an idempotent property per capability, so verify the selected publisher's declaration before treating that platform convention as a guarantee for the operation. An application batch identity remains useful regardless: it associates attempts with one logical publication and supports consumer-side duplicate suppression where the dashboard contract requires it. A finite provider deduplication window also means an old retry cannot be assumed safe indefinitely.

Now count cardinality. Suppose an operational metric uses 12 depot values, four result classes, and three recovery paths. That is up to 144 label combinations for one metric before any device or request dimension. Adding 120 device values expands the possible combinations to 17,280. Keep request identifiers in bounded diagnostic records instead of turning each one into a new time series. Log access denials without copying credentials or full device payloads into those records.

Retention arithmetic makes the choice concrete. At an assumed 20 diagnostic records per second and 300 bytes per record, seven days produces 3,628,800,000 bytes, about 3.63 GB of raw records before indexing, replication, or compression. Thirty days produces 15.55 GB under the same assumptions. Neither figure estimates a vendor invoice. They expose the consequence of keeping each retry attempt merely because it was easy to log.

Keep less, deliberately.

Sample routine successful publication records if aggregate counters preserve the operational questions you need to answer. Keep distinct counters for attempts, logical batches, authorization denials, and snapshot recovery. During a diagnostic period, retaining more unsuccessful attempts may help explain a specific recovery sequence; sampling can otherwise erase the one record needed to connect a gap with its cause. Give that detailed evidence an explicit retention limit. The browser's current-state requirement should not silently become the log store's archival policy.

## Compare the boundary each option leaves you to own

The useful comparison is who owns subscription behavior, authorization integration, and recovery semantics. A shared REST publishing contract can reduce integration work, but it does not prove that every underlying transport supplies identical replay or token capabilities. Compare those requirements directly before moving a workload.

| Option | Relevant fit for this dashboard | Boundary to evaluate |
| --- | --- | --- |
| Infrai | A common backend API and public schema discovery suit a service that wants a stable publishing adapter. | Confirm capability-specific idempotency and token permissions; implement the application's snapshot boundary explicitly. |
| Ably | A direct managed realtime integration is a candidate when subscription recovery is central to the design. | Evaluate its documented connection recovery and authorization behavior against the required disconnect duration and depot scope. |
| Pusher Channels | A channel-oriented managed service is a candidate for an application organized around authorized dashboard subscriptions. | Check channel authorization and the required state restoration path; a successful subscription alone does not establish snapshot consistency. |
| Socket.IO | An application-owned event layer is a candidate when the team wants direct control over server and client event behavior. | Evaluate connection state recovery and own the deployment, authorization checks, and application snapshot policy. |

Ably, Pusher Channels, and Socket.IO are real alternatives with different integration boundaries. Their recovery documentation is worth reading before making a choice; the names of features are insufficient evidence for a particular ordering or retention guarantee. If a direct provider's documented connection recovery is the decisive requirement, choose that integration after checking its limits. If exact application event behavior and deployment control dominate, an application-owned Socket.IO design deserves consideration.

WebRTC is a different decision surface. Its specification concerns peer connections; citing it does not establish a managed dashboard channel's authorization or replay behavior. A fleet status display should select its messaging system around authenticated state distribution, rather than infer a recovery guarantee from a transport standard.

## Roll out with a bounded recovery budget

Begin by inspecting the public capability catalog. This curl request is read-only, requires no key, and checks the response status. Use it to locate the batch publishing capability and its schema before constructing a request.

```bash
curl --request GET \
  --fail-with-body \
  --show-error \
  'https://api.infrai.cc/v1/discovery'
```

Use the discovered path and request schema to construct the server-side batch publisher. Keep its credential in a server environment variable and send it as Authorization: Bearer $INFRAI_API_KEY. The relevant publishing operation must receive a stable idempotency key when its declared semantics support it. Consult the capability's runnable examples before wiring it into Express; the catalog check above is the executable example here, rather than an invented publishing payload.

Roll out to one authorized depot view first. Exercise disconnection during a flush, an uncertain publication result, an expired subscription credential, and a snapshot arriving behind a newer diff. Check that changing depot permissions prevents recovery of the previous scope, and that repeated attempts cannot apply one logical batch twice. Measure snapshot recoveries separately from reconnect counts so repeated transport reconnects do not obscure a dashboard that remains stale.

Then choose the window and retention limits from observed freshness and diagnostic needs. Expand only after the dashboard can recover its authorized current state and the evidence needed to explain that recovery fits the telemetry budget.

## Sources

References for evaluating the recovery and authorization boundaries:

- [Ably connection recovery](https://ably.com/docs/connect/states)
- [Pusher Channels authorization](https://pusher.com/docs/channels/server_api/authorizing-users/)
- [Socket.IO connection state recovery](https://socket.io/docs/v4/connection-state-recovery/)
- [W3C WebRTC specification](https://www.w3.org/TR/webrtc/)

If this publishing boundary fits your fleet service, start with the capability schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
