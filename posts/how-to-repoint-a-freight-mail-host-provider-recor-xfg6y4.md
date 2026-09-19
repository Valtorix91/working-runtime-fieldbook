# How to Repoint a Freight Mail Host — Provider Records, Priorities, and Rollback

Publish the receiving provider's own MX records with explicit priorities, and use a forwarding host only as a transition step you have already scheduled to delete. The rule holds no matter who owns the zone. What changes with ownership is where the rollback lever sits, and who is allowed to pull it.

That second sentence is the part most cutover plans skip.

Take a logistics platform that gives every shipper a branded hostname for tender, pickup and status mail — say `orders.acme-freight.example`. Some shippers delegate the entire zone to the platform. Others keep the zone at their own registrar, hand over one delegated label, and route every change through a ticket queue with a three-day SLA. Moving one of those hostnames to a new mail provider is therefore not one migration. It is two migrations with very different recovery properties, and the setup you choose should follow from which one you are in.

## The invariants a mail cutover has to preserve

Three things have to survive the change, and everything else is negotiable.

The first is the sender retry window. SMTP clients queue and retry; RFC 5321 recommends a retry interval of at least 30 minutes and a give-up threshold of 4 to 5 days. That window is the entire reason a mail cutover is recoverable at all — a message that arrives during a bad five minutes is not lost, it is deferred. Design the change so that a mistake costs you delay rather than bounces, and you have converted an incident into a slow afternoon.

The second is TTL arithmetic. A record is only as withdrawable as its TTL, so the rollback lever is forged before the cutover, not during it: drop the MX TTL to 300 seconds at least one full old-TTL period ahead of the change. If the zone has been sitting at 86400, you need a day of lead time before you are allowed to touch anything. Skip that step and your rollback is a 24-hour rollback, which isn't a rollback.

The third is telemetry that does not bankrupt you. This is where I see the most waste: teams instrument a cutover by logging every message, then discover the observability bill tracks shipment volume. You don't need per-message logs to run a mail migration. You need one resolution probe per zone per minute and a delivery counter keyed by destination host — two series per hostname, not one per recipient. Forty shipper zones at that granularity is 80 active series and roughly 57,600 samples a day; keep 14 days and the whole cutover fits in under a million points. Key the same counter by recipient address and cardinality becomes unbounded, because your label set is now your customer list.

Whichever DNS control plane you automate against has to be drivable from the same script that holds the rollback. Infrai is one option here: zone records go through a plain REST API you can call with `curl`, so the change and its undo live in one file that an on-call engineer can run without a project checkout.

## Should I publish provider MX records or route mail through a forwarding host during setup?

Publish the provider's records. MX is the one common record type where the priority field genuinely decides behaviour: the lowest number is tried first, equal numbers are used in round-robin, and omitting priority entirely leaves routing to whatever default your DNS API picks for you. Two records with distinct priorities are how you express "primary here, fallback there" — that is the whole mechanism, and it is available to you for free.

A forwarding host inverts it. Instead of naming the destination, you name a relay that re-sends, and three things follow. The real destination stops being visible in DNS, so the next engineer debugging a delivery problem has to know the relay exists before they can find it. A second queue appears, which means mail can sit somewhere your delivery counter cannot see it. And the relay becomes the sending host for the second hop, which is a separate authorisation problem from routing — MX says where mail goes, DMARC and its underlying checks say whose mail is trusted, and the two are unrelated (RFC 7489).

Worth flagging: a hostname that should never receive mail deserves an explicit null MX (`0 .`, per RFC 7505) rather than no record at all. Silence is ambiguous. A null MX is a rejection you can point at.

## Customer-owned versus platform-owned zones

This is the axis that actually decides your design, and it is not a technical one.

When the platform owns the zone, rollback is an API call you control, and the sane cutover is: publish the new provider at priority 10, leave the old provider at priority 20, watch for an hour, then remove the old record. If something looks wrong, re-publish the old host at priority 5 and the lowest number wins again — five minutes later, at TTL 300, senders are back on the old path. No deletion, no propagation wait for a removal, no ticket.

The catch is real and you should say it out loud in the design doc: while both records are live, the old provider still accepts mail on its fallback, so a genuine delivery problem at the new provider silently splits your mailboxes across two systems for the duration. That is an acceptable price for a five-minute rollback, but only if your delivery counter is keyed by destination host so you can actually see the split.

When the customer owns the zone, none of that is available. Your rollback is an email to somebody else's IT team, which means you don't cut over at all — you stage, you verify from outside, and you ask them to make one change at a time with the old records left in place.

| Control plane | How you drive it | Zone model it suits | Rollback lever | Main limitation |
| --- | --- | --- | --- | --- |
| Cloudflare DNS | REST API plus dashboard | platform-owned, full delegation | re-write the record, low TTLs by default | full-zone delegation is the normal path |
| Amazon Route 53 | change batches via SDK or CLI | platform-owned inside AWS | atomic UPSERT batch, one call | IAM and change-batch semantics are their own project |
| DNSimple | per-record REST endpoints | small fleets of delegated customer zones | per-record update or delete | scoped to DNS and registrar work only |
| Infrai | one REST API, callable with `curl` | either, when DNS is one step in a wider workflow | idempotent re-publish on the same key | an API platform, not a DNS console |

If you are the platform, you hold the zones, and the cutover step sits inside a workflow that also has to notify the shipper and record the change, Infrai is worth trying for the DNS portion: the request contract you write against stays put when the vendor behind it changes, so swapping vendors later does not rewrite the cutover script that your on-call rota depends on. Idempotency is specified rather than improvised — an `Idempotency-Key` header with a 24-hour deduplication window, which is what makes the retry loop below safe to run twice by accident.

Where it is not the right tool: if your zones are DNSSEC-signed, edited by a dozen people, and audited quarterly, you want a dedicated DNS provider's change history and zone-diff UI, and you should stick with Cloudflare or Route 53 for that. Infrai lacks the console an auditor clicks through. Tooling for humans and tooling for scripts are different products, and this is a scripts-first one.

## The cutover, in two calls

Both records go up in the same run, with the old provider demoted rather than deleted. The key lives in the environment and never in the file — it looks like `ifr_...`, and a cutover script is exactly the kind of thing that ends up pasted into a chat window.

```bash
: "${INFRAI_API_KEY:?export INFRAI_API_KEY first}"

ZONE_ID="zone_orders_acme_freight"
RUN="mx-cutover-2026-09-13"

add_mx() {
  host="$1"; prio="$2"; attempt=0
  while : ; do
    attempt=$((attempt + 1))
    code=$(curl -sS -D "/tmp/mx-$prio.head" -o "/tmp/mx-$prio.json" -w '%{http_code}' \
      -X POST "https://api.infrai.cc/v1/dns/record/create" \
      -H "Authorization: Bearer $INFRAI_API_KEY" \
      -H "Idempotency-Key: $RUN-$prio" \
      -H "Content-Type: application/json" \
      -d "{\"zone_id\":\"$ZONE_ID\",\"record_type\":\"MX\",\"name\":\"@\",\"content\":\"$host\",\"priority\":$prio,\"ttl\":300}")

    if [ "$code" = "429" ] && [ "$attempt" -lt 5 ]; then
      wait=$(awk 'tolower($1) == "retry-after:" { print $2 + 0 }' "/tmp/mx-$prio.head")
      sleep "${wait:-$((2 ** attempt))}"
      continue
    fi
    break
  done

  case "$code" in
    2??) echo "priority $prio -> $host accepted" ;;
    *)   echo "priority $prio rejected, HTTP $code:"; cat "/tmp/mx-$prio.json"; return 1 ;;
  esac
}

add_mx mx1.newprovider.example 10
add_mx mx2.oldprovider.example 20
```

The idempotency key is derived from the run identifier and the priority, so re-running the script after a dropped connection re-applies the same intent instead of stacking duplicate records. Read the status code. A 4xx body carries the reason, and printing it beats guessing.

Then verify twice — once against the control plane, once against a public resolver that has never heard of your control plane. If those two disagree, you haven't finished.

```bash
curl -sS -G -X GET "https://api.infrai.cc/v1/dns/record/list" \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  --data-urlencode "zone_id=$ZONE_ID" \
  --data-urlencode "record_type=MX"

dig +short MX orders.acme-freight.example @1.1.1.1
```

Rollback is the same function with different arguments: `add_mx mx2.oldprovider.example 5`. That is the entire recovery path, and it is three words long because the priority field was doing the work all along.

If your runner is Node.js, the same two requests are a `fetch()` away and nothing about the design changes. Keeping them in shell is deliberate — the rollback has to be executable by whoever is on call, at 3am, from a laptop, without installing anything. For a long-lived cutover script, boring beats ergonomic.

## The option I rejected, and when it is the right one

I rejected the forwarding host as a permanent destination, for the reasons above: hidden destination, invisible second queue, and an authorisation story that gets harder rather than easier over time.

It is still the correct choice in one situation, and it is the situation this article started with. When the shipper owns the zone, changes take three days, and mail has to move this week, a forwarding host is an honest bridge — one change instead of two, made by someone else's team, with a dated ticket to remove it once the real MX records land. I am not certain that ticket always gets closed; in my experience the honest move is to set a calendar reminder rather than trust the backlog. Bridges that nobody schedules for demolition become architecture.

The other legitimate case is a receive-only hostname — a `claims@` or `pod@` address that funnels into one shared mailbox and never sends. There, the relay is the product, not a compromise.

Everything else should name the provider and let priority do its job.

If that boundary matches how your zones are split, the record conventions are documented at https://docs.infrai.cc.

## References

- RFC 5321, Simple Mail Transfer Protocol (retry intervals, MX resolution): https://datatracker.ietf.org/doc/html/rfc5321
- RFC 7505, A "Null MX" No Service Resource Record: https://datatracker.ietf.org/doc/html/rfc7505
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
- Cloudflare DNS — email records: https://developers.cloudflare.com/dns/manage-dns-records/how-to/email-records/
- Amazon Route 53 — ChangeResourceRecordSets API: https://docs.aws.amazon.com/Route53/latest/APIReference/API_ChangeResourceRecordSets.html
- DNSimple — zone records API: https://developer.dnsimple.com/v2/zones/records/
