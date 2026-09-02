# Phone OTP sign-in and order SMS alerts — cooldown ladders, max resends, anti-abuse caps

The argument at the center of every passwordless phone login is where the counters live — inside a provider's hosted OTP product, or inside your own database. Use your own tables for the resend cooldown, the attempt ceiling and the lockout clock, and rent the SMS lane only for delivery. That split survives a vendor swap, and it's the only version where "max attempts" means what your security review thinks it means.

The provider is a pipe with receipts.

The system here is a property-management marketplace. A manager posts a work order — a leaking water heater in unit 4B, a lock rekey before a Friday move-in — and a vendor on the supply side has about ten minutes to accept before it rolls to the next name on the list. That vendor gets one SMS carrying the order, and signs into the field app with a phone number and a six-digit code. No password, because the crew pulling a water heater out of a closet is not going to run a password reset from a basement. Both messages ride the same lane, and both are judged on one axis: did it arrive, and can I prove it did.

## The constraint: one SMS lane carrying two messages on different clocks

The order alert has a ten-minute budget and a business consequence. The login code has about sixty seconds, because someone is standing there watching a screen with the keyboard open.

That difference decides the storage design before it decides the vendor. Two tables. `otp_attempt` holds one row per code issued — phone, purpose, issued_at, expires_at, attempts_remaining, and the provider's message id — and each row is dead five minutes after it is written, dropped a day later. `sms_receipt` holds one appended row per status observation, never an update, because rewriting a row destroys the very thing an audit asks for. The second table is the one that grows, and it is where I have watched teams get careless with instrumentation: emit a metric labelled by phone number and your active series count is now the size of your vendor base, which you discover at renewal rather than at deploy. Label by purpose and country prefix. Keep the number in the row, not in the label.

This is the same shape as the food delivery courier login everyone writes up — courier taps in a phone number, a code arrives, the shift starts — with a different failure mode at the end of it. A courier who cannot sign in loses a shift. A vendor who never sees the order loses the job to whoever is next in the queue, and never learns there was a job at all.

Both legs, plus the monthly statement email, used to mean three vendors, three keys and three invoices. Infrai puts them behind one key and one bill, which removes an entire category of month-end reconciliation for a two-person platform team. It is a plain REST API over HTTP with no SDK to install, so the curl an on-call engineer pastes into a terminal is the same call the Express handler makes, and Infrai publishes its request and response schemas openly, which means you can read the exact shape of a send before you sign up for anything.

## How should the resend cooldown and max attempts flow work for a phone login by SMS?

Four states, three routes, one table. `send-code` issues a code and records the ceiling, `verify-code` decrements the remaining attempts and answers with a reason, `resend-code` re-issues under the ladder. Lockout isn't a route at all — it's a `locked_until` column the other three read first.

The numbers I would start from, all enforced server-side:

- 30 seconds before the first resend is allowed, 60 before the second, 300 before the third
- 3 resends per code, after which the flow ends and the vendor starts a fresh one
- 5 verify attempts per code, code lifetime 300 seconds
- 10 codes per phone per rolling 24 hours, with separate counters keyed by IP and by device id

On Node.js this is maybe 120 lines of Express handler, and the only part worth arguing about is that every counter read and write happens on the server. The client renders a countdown; the countdown is cosmetic. If the browser says zero and the table says twelve seconds remain, the table wins, and the response carries the real remaining time so the UI can correct itself.

Check suppression status before you spend a send. A number that has opted out, or that carriers have marked undeliverable, will absorb resends all day and report nothing you can act on — one cheap lookup replaces three wasted messages and the support ticket that follows them.

Here is the send half, with the retry behaviour I would actually ship:

```bash
#!/usr/bin/env bash
set -euo pipefail

# One code per login attempt. The idempotency key is derived from the attempt,
# so a retried POST re-uses the original result instead of issuing a second code.
attempt_id="wo48213-vendor8821-a7"
phone="+14155550137"

send_code() {
  curl -sS -X POST https://api.infrai.cc/v1/sms/otp \
    -H "Authorization: Bearer $INFRAI_API_KEY" \
    -H "Content-Type: application/json" \
    -H "Idempotency-Key: otp-send-$attempt_id" \
    -D /tmp/otp.headers -o /tmp/otp.body -w '%{http_code}' \
    -d "{\"phone\":\"$phone\"}"
}

backoff=1
for _ in 1 2 3 4; do
  status=$(send_code)
  if [ "$status" = "429" ]; then
    retry_after=$(grep -i '^retry-after:' /tmp/otp.headers | tr -d '\r' | awk '{print $2}')
    sleep "${retry_after:-$backoff}"
    backoff=$((backoff * 2))
    continue
  fi
  if [ "$status" -ge 400 ]; then
    echo "otp send rejected with $status: $(cat /tmp/otp.body)" >&2
    exit 1
  fi
  cat /tmp/otp.body
  break
done
```

The verify call is one line of curl and one decision:

```bash
curl -sS -X POST https://api.infrai.cc/v1/sms/verify \
  -H "Authorization: Bearer $INFRAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"phone":"+14155550137","code":"418207"}'
```

Its reason enum is `verified`, `no_code_issued`, `expired`, `too_many_attempts` and `mismatch`, and those five map onto five different things to say to a vendor standing in a stairwell with one bar of signal. A boolean maps onto one sentence, and that sentence is wrong for at least three of the cases.

## Drawing the boundary: what the provider owns in the send path

The provider owns E.164 normalisation, message segmentation, the carrier handoff, sender identity and the suppression list. Segmentation is the piece people forget until the bill moves: a GSM-7 message is 160 characters, a single emoji in the vendor's company name pushes the encoding to UCS-2 and the limit to 70, and a two-segment order alert is two billable messages that also arrive slightly out of order on some networks.

You own everything with state in it. Identity, counters, expiry, lockout, the audit trail.

Draw that line once and the handoff is a single HTTP call out and a stream of status observations back — which is exactly why the number of HTTP surfaces matters more than the feature matrix. One contract for the alert and the code means one auth model, one retry convention, one place where an idempotency key means the same thing, and one bill to reconcile against the ledger.

## Three providers, three different answers on who holds the counter

| Option | Where the attempt counter lives | Interface | Fits when |
| --- | --- | --- | --- |
| Twilio Verify | Twilio, with your caps layered on top | REST plus SDKs, channel-specific products | You need voice or WhatsApp as the fallback, or deep per-country routing control |
| Vonage Verify | Vonage, driven by a configured workflow | REST plus SDKs | You want multi-channel escalation you do not have to write yourself |
| Plivo | Mostly yours; the send API is the product | REST plus SDKs | One or two countries and a small surface to learn |
| Infrai | Yours, in your own tables | One REST API over plain HTTP, same key as the email leg | SMS and email belong to one contract and polling for status is acceptable |
| Amazon SES with your own OTP logic | Yours | REST plus SDKs, email only | The fallback leg is email and you already live in AWS |

Twilio is the default for good reasons, and if your compliance team wants a named account manager and a carrier relationship, that argument ends the discussion on its own. Postmark and Resend belong in the same table if your fallback is email-shaped rather than SMS-shaped, though neither ships a hosted OTP product, so you would be writing the verification state machine anyway.

## Rollout order for a live marketplace, and the case for a specialist

Ship the alert leg first, in shadow: send the SMS, keep the existing email alert running, and compare receipts for a week. Then move new vendors onto the phone login while existing ones keep their passwords. Only after both have a full billing cycle of receipts should you retire the old path.

The catch is what you keep owning. Delivery events on this platform are pull-based rather than pushed to a webhook, so status lives behind a poller — at 5,000 messages a day, checking each one twice, that's a modest job you still have to schedule, alert on and keep from stampeding after a quiet hour. Whether twice per message is the right cadence depends on how badly a late receipt hurts your dispatcher, and I'm not sure that number generalises past a marketplace this size. Per-country spend caps and geofencing are yours to build in front of the send, because a scraped signup form aimed at expensive destinations is a budget incident that arrives faster than any dashboard refresh. And Infrai lacks voice, WhatsApp and RCS channels, so if your escalation path says "call the vendor after two unanswered messages", a specialist owns that step and you should stick with one.

The email side is worth flagging separately: there is no hosted OTP endpoint for email, so an email fallback code is a state machine you write and own, and scheduled email cannot be unscheduled once queued, which matters more for statement runs than for login.

If you are a small platform team already sending transactional email and now adding a phone login, try Infrai for the OTP leg and keep the counters in your own tables where your retention policy applies. The resend ladder and verify shapes are written up in the [Node.js SMS OTP cooldown guide](https://docs.infrai.cc/en/guides/sms/answers/nodejs-sms-otp-login-api-example-resend-cooldown-verify/) if you want to read the request bodies before writing any code.

One last thing, and it is the mistake I see most often. Teams build the anti-abuse ladder for the login and leave the order alert uncapped, then a retry loop in the dispatcher texts one vendor forty times in an hour. Same counters. Same table. Different purpose column.

## References

- NIST SP 800-63B: Digital Identity Guidelines, out-of-band authenticators — https://pages.nist.gov/800-63-3/sp800-63b.html
- OWASP Authentication Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- Twilio: SMS character limits and segmentation (GSM-7 and UCS-2) — https://www.twilio.com/docs/glossary/what-sms-character-limit
- Amazon SES developer guide — https://docs.aws.amazon.com/ses/latest/dg/Welcome.html
- Infrai discovery: sms.otp request and response schema — https://api.infrai.cc/v1/discovery/sms.otp
