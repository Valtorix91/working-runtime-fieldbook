# Node.js Authentication Telemetry: Budgeting SMS Polls and Email OTP Template Ownership

A media service that emails generated reports needs a clear owner for its login templates before it adds SMS-to-email OTP fallback. The observability budget then follows three quantities: challenge volume, status checks per challenge, and retained bytes per check. Count those before choosing how much delivery detail to keep.

TL;DR: For a Node.js SaaS service using polling without webhooks, bound each challenge's polling work, make fallback an explicit authentication policy, and keep template ownership with the team responsible for that policy. Retain compact state transitions; sample routine delivery diagnostics. An email attachment belongs to the report workflow, never to the login-code message.

Consider an illustrative planning workload of 100,000 login challenges per day, six recorded status checks per challenge, and 800 bytes per stored record. Those assumptions yield 480,000,000 bytes per day, or 14.4 GB over 30 days in decimal units, before indexes, replicas, and other storage overhead. These are hypothetical inputs, not measured traffic or a provider's billing model. They expose the dominant term in this example: repeated status records. Message charges, polling requests, ingestion, and storage remain separate budget lines.

## What should SMS-to-email OTP fallback retain for a 2FA login?

Delivery evidence and authentication evidence answer different questions. A delivery status can inform whether another channel should be offered. It cannot establish that the person entering a code controls the intended account. With no webhook consumer, design the worker to read status through the selected transport's documented interface and translate the result into a small internal state vocabulary. Whether that interface exists, what its statuses mean, and which limits apply must be verified during selection; no particular polling contract is assumed here.

Keep states few.

For example, a proposed internal model could distinguish a pending challenge, an offered fallback, a verified challenge, and an expired challenge. These are application decisions, not claims about a messaging API. A network timeout leaves delivery uncertain; it should not silently authorize an email challenge. Likewise, an SMS that appears delayed may still arrive after the user requests email. Define whether fallback replaces the active code, and enforce that decision atomically when verification occurs. A replacement policy should ensure that an old code cannot later complete the superseded challenge.

OWASP's password-recovery guidance provides useful adjacent controls: tokens should be securely generated, stored securely, single use, and time limited, with protections against excessive requests and guessing. Its scope is recovery, so those controls do not by themselves establish that email is an appropriate second factor for a particular login. Treat permission to fall back as a separate security decision. For accounts where the required assurance excludes email recovery, the outcome of exhausted polling is an explicit recovery path, not an automatic downgrade.

## Count retained bytes before adding another status check

The planning equation is challenge count × recorded checks × bytes per record × retention days. A request budget needs an additional bound on total checks, including retries after errors. If six is the intended maximum, retries must consume that same allowance. Otherwise the telemetry estimate describes ordinary traffic while the failure path can spend without a ceiling.

The following comparison holds the illustrative 100,000 daily challenges and 30-day retention constant. It isolates changes to application records; it does not estimate a complete observability bill.

| Proposed recording policy | Assumed daily bytes | Retained bytes over 30 days | Diagnostic consequence |
| --- | ---: | ---: | --- |
| Six 800-byte status records per challenge | 480 MB | 14.4 GB | Every planned check remains available |
| Two 300-byte transition records per challenge | 60 MB | 1.8 GB | Intermediate polling history is absent |
| Two transition records plus a 10% sample of full check records | 108 MB | 3.24 GB | Detailed history exists for sampled challenges only |

The second row assumes two transitions per challenge for arithmetic. An implementation must count its actual transitions, including expiry and fallback, rather than force them into that estimate. The third row assumes the same six-check workload and retains an additional 48 MB of sampled diagnostics each day. None of these figures includes backups or indexing.

Reduce repetition first.

Shortening retention changes the time horizon for investigation. Reducing routine status duplication changes the evidence collected for each challenge. These choices have different consequences, even when their storage totals happen to match. A compact transition record should preserve the decision, its timestamp, a bounded reason category, and the template revision needed to explain the message that was issued. Exclude the OTP, rendered message body, recipient address, and generated report contents from routine diagnostic records.

Cardinality deserves its own budget. A proposed metric with two channel values, four state values, and three outcome values has up to 24 combinations before other dimensions. Adding an unconstrained account identifier multiplies that space by the number of distinct accounts represented. Keep account, recipient, and challenge identifiers out of metric labels. If correlation requires an opaque challenge reference, confine it to access-controlled records with an explicit expiry policy; replacing an address with an identifier does not remove the need to govern its retention.

## Template ownership separates access from the report attachment

The media workflow has two distinct messages: a login challenge and an email carrying a generated report. Give the authentication owner authority over the OTP wording, code lifetime description, and fallback explanation. Give the report owner authority over attachment presentation and editorial copy. A shared rendering service can support both, but its convenience should not permit a report-template change to alter authentication behavior.

For selection, ask who can review a template revision, how an application pins or records that revision, and how a rollback restores the previously approved content. These are acceptance criteria to test against the chosen implementation. They are especially consequential if templates live outside the application repository: deployment can then change one side of the contract while leaving the other untouched. Record a bounded revision field with issuance events so an investigation can identify the approved wording without retaining every rendered email.

One concrete failure to test is a template that says the previous code remains valid after fallback even though the application has replaced it. Another is a report email that encourages the recipient to reply with a login code. Both create confusion at the security boundary. Test rendered messages against policy, including expiry wording and the absence of report attachments in OTP messages. Successful rendering alone is insufficient.

For US recipients, the FTC's CAN-SPAM guide distinguishes commercial messages from transactional or relationship messages and discusses how mixed content affects the primary-purpose analysis. Keep promotional copy out of authentication templates and review report messages according to their actual content. The cited sources do not settle EU legal obligations, cross-border processing, or a particular service's retention requirements. Resolve those with a documented jurisdictional review before deployment; a US transactional classification is not an EU compliance decision.

## Bound the worker and test the uncertain outcomes

A polling worker should stop when the challenge expires, verification succeeds, fallback supersedes the original challenge, or its check allowance is consumed. Store enough shared state that restarting a Node.js process does not reset that allowance. Choose polling intervals from the user-visible waiting policy and documented transport limits, then test the resulting request volume under concurrent logins. Do not infer permission to poll from a successful send request.

Deployment tests should exercise a status timeout, a delayed SMS arriving after email issuance, simultaneous verification attempts, an exhausted polling allowance, and a template rollback. Check both the authentication outcome and the retained evidence. In particular, concurrent attempts should not produce two successful consumptions of a single-use challenge. A delivery timeout should remain distinguishable from an authentication rejection in operational counters. That distinction makes incident triage possible without recording the code itself.

Sampling requires a deliberate exception policy. Preserve the compact security-relevant transitions for every challenge under the chosen retention policy, and sample verbose delivery diagnostics at the challenge level so a selected trace includes its related checks. Retaining every error diagnostic may help explain uncommon failures, but a widespread failure can make that exception the largest stream. Give it a separate volume bound and a counter for discarded diagnostics. Otherwise the incident changes both the load and the assumptions behind the budget.

## What the retention decision gives up

The proposed operating choice is compact transition evidence for each challenge, bounded polling, and sampled routine diagnostics, with template revisions owned by the authentication team. Set the retention duration after establishing investigation needs and jurisdictional requirements. The 30-day horizon above is only a calculation input. It is not a compliance rule.

Deliberately stop keeping repeated unchanged statuses, complete rendered OTP messages, and report attachments in the authentication log. The trade-off is reduced forensic detail: for an unsampled challenge, support may be unable to reconstruct the exact sequence of intermediate delivery observations or inspect the exact rendered message. After transition records expire, even that decision history is gone.

Accept that loss explicitly.

Retain aggregate counters for trend analysis under their own policy, monitor dropped diagnostics, and verify that the remaining records can explain why fallback was offered. The useful endpoint is an auditable authentication decision with a known evidence window. Keeping every delivery observation indefinitely does not resolve template ownership or establish who authenticated.

## References

The OWASP recovery guidance supports the token-lifecycle and abuse-prevention controls discussed above. The FTC guide supports the US message-classification discussion. The workload, recording policies, cardinality dimensions, and state model are explicitly proposed examples; neither source prescribes those quantities or a polling implementation.

## Further reading

- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- FTC CAN-SPAM Act compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
