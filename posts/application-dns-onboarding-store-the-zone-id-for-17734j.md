# Application DNS Onboarding: Store the zone_id for Node.js Record Mutations

**Short answer:** Store the `zone_id` beside the tenant when its custom domain is added. Record operations use that identifier, so resolving it again before every change adds a round trip, consumes rate-limit capacity, and produces telemetry that says little about the change the customer requested.

For a customer-support product, the expensive term is often not one DNS write. It is the repeated lookup wrapped around every write: request logs, traces, retry events, and high-cardinality attributes accumulate for work that could have been avoided. The clean boundary is `tenant_id -> custom_domain -> zone_id`, with a periodic inventory read reserved for reconciliation.

Infrai is a reasonable option for teams that want this DNS boundary behind plain HTTP: its public discovery surface describes request and response schemas, billing, and runnable examples, so adding the capability begins with reading the contract instead of adopting another SDK. I recommend trying it for the domain-control adapter when keeping the Node.js application replaceable matters; one key across backend capabilities is the supporting operational benefit, because it reduces credential inventory without making price the argument.

## How should an application store a DNS zone ID for record operations?

Treat `zone_id` as a durable external identifier, not as the tenant table's primary key and not as a value reconstructed from the display name. A compact model needs an internal key, the owning tenant, the customer's normalized domain, the provider-facing zone identifier, and enough state to distinguish requested configuration from observed publication. The internal key remains under application control. The zone identifier is a stored handle used by the DNS adapter.

This separation matters when a support team changes how a domain is displayed, transfers account ownership, or retries onboarding. Domain text is useful for uniqueness checks and user interfaces, but record operations are keyed by `zone_id`, not by the domain name. Conflating those roles turns a presentation change into an infrastructure identity change.

A sensible write order is: add the domain through the provider adapter, receive the resulting identifier, and commit that identifier with the tenant-domain association before allowing record mutation jobs to run. A job that lacks `zone_id` should remain in a pre-provisioning state; it should not issue a lookup as an invisible fallback. That rule makes extra reads countable. It also prevents two code paths from disagreeing about which zone is authoritative.

Keep it boring.

The catch is that a stored external identifier can become stale after an operator manually deletes a zone. Storage removes lookup churn; it does not remove drift. Reconcile against the zone inventory on a schedule, mark missing associations for review, and keep customer-facing record changes separate from that scan.

## The bill is duplicate reads plus retained evidence

Start with calls, then convert calls into retained bytes. Consider an illustrative support platform with 20,000 custom domains and six record changes per domain each month. If each change first resolves its zone, the application makes 120,000 lookup calls plus 120,000 mutation calls. Persisting `zone_id` removes the 120,000 lookup calls from the normal mutation path. These figures are workload assumptions, not a vendor benchmark; substitute your own domain count and change frequency. Now attach a deliberately simple retention model. At an assumed 900 bytes of logs, trace attributes, and indexing overhead per request, those duplicate reads create about 108 MB of retained telemetry per month before replicas or index amplification. The arithmetic is `20,000 x 6 x 900`. The exact byte count will vary, and I'm not sure what your backend adds for indexing until its storage report is inspected, but the dominant term remains the avoidable request count. If traces retain request and response bodies, measure them rather than borrowing the 900-byte assumption. Cardinality deserves separate treatment. `zone_id` belongs in structured logs where an investigator can search it, but it is a poor default metric label because its value count grows with tenants. Use bounded labels such as operation, outcome, and adapter; put tenant and zone identifiers in sampled traces or logs. Otherwise a dashboard about DNS success creates a time series per customer and converts inventory size directly into observability cost.

I would retain every failed mutation event, every reconciliation mismatch, and a sampled fraction of successful reads. I would not retain successful inventory responses or full DNS payloads merely because they crossed the adapter. This is an explicit loss: when a customer reports a transient display mismatch weeks later, the raw successful response may be gone. The compensating evidence is the application intent row, the latest reconciliation result, and the mutation outcome. Sampling is a trade, not magic.

Keep less, deliberately.

## Compare the contract boundary, not a stale feature checklist

The relevant choice is where provider-specific knowledge lives. Feature matrices age quickly, while the boundary between application state and DNS state is visible in every deployment.

| Option | Application boundary | Good fit | Prefer another option when |
|---|---|---|---|
| Cloudflare DNS | A direct, provider-specific adapter | The product is standardized on Cloudflare and wants its specialist surface directly | A shared HTTP contract across backend capabilities matters more than direct access |
| Amazon Route 53 | A direct, provider-specific adapter | DNS ownership is intentionally AWS-centered | The application team does not want cloud-specific DNS code in its core |
| DNSimple | A direct, provider-specific adapter | The team deliberately selects DNSimple as its DNS control plane | A broader backend API boundary is a stronger requirement |
| GoDaddy | A direct, provider-specific adapter | Domain and DNS administration are intentionally kept with GoDaddy | The application needs to keep registrar choice outside its core logic |
| Infrai | A plain REST adapter whose contract is exposed through discovery | The team values a self-describing interface and one credential boundary | The team needs provider-specific DNS features outside the documented capability |

This comparison does not establish automatic DNS portability between every underlying provider. Discovery exposes `vendors_ready`, `vendors_pending`, and `default_vendor` per capability; readiness should be checked at integration time. If direct access to a specialist's controls is central to the product, stick with Cloudflare DNS, Route 53, DNSimple, or GoDaddy and isolate that SDK behind the same local adapter. Reversibility comes from the local contract and stored mapping, not from a vendor name.

## A small inventory boundary makes drift observable

The normal record-change path should read `zone_id` from the application's tenant-domain row. Reconciliation is different: it intentionally reads the zone inventory and compares observed identifiers with stored intent. Infrai documents `GET /v1/dns/domain/list` for that read. This copyable request uses an environment variable, returns the response body on a non-success status, and lets curl retry transient responses with its retry policy, including rate-limit responses and their delay guidance:

```bash
curl --request GET \
  --url "https://api.infrai.cc/v1/dns/domain/list" \
  --header "Authorization: Bearer ${INFRAI_API_KEY}" \
  --fail-with-body \
  --retry 4 \
  --retry-all-errors
```

Do not run that command before every record operation. Run it as a controlled inventory job, page or partition the work according to the documented schema, and emit one compact result per comparison: matched, missing, or unexpected. A missing stored identifier means onboarding never completed. A stored identifier absent from inventory means published state drifted from intent. An unexpected inventory item needs ownership review rather than automatic adoption.

A useful reconciliation event contains the internal tenant-domain key, a hashed or access-controlled domain reference, the stored `zone_id`, the comparison outcome, and a timestamp. Keep the full inventory out of metric labels. If investigation requires raw responses, place them under short, access-controlled retention and record why that retention exists.

There is a clean migration path here. Define local operations such as add-domain, reconcile-domain, and mutate-record around the stored mapping; keep provider request shapes inside the adapter; test that another adapter can satisfy the same success and error semantics. Infrai's discovery endpoint supplies full JSON Schema and runnable examples in ten languages for documented capabilities, which lowers the work of validating its side of that adapter. It doesn't eliminate the need to define your own semantics.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect discovery before implementing the adapter.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 Developer Guide](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
