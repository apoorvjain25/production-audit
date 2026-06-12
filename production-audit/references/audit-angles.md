# Audit angles — the discovery-pass lens catalog

Each discovery pass takes ONE lens from this catalog and sweeps the whole inventory through it. Never run the same lens twice. Convergence ("two consecutive passes find nothing new") only means something if the passes are genuinely diverse — ten subsystem sweeps are one angle, not ten.

Pick the next lens by asking: *which framing has not yet had a chance to make a defect visible?*

## 1. Subsystem sweep
Walk one subsystem at a time (auth, billing, search, notifications, import/export, admin, …) end to end. Read the entry points, trace to storage and back. Best first pass — it builds the mental map the other lenses need.

## 2. Attack-class
Pick one vulnerability class and hunt it everywhere:
- IDOR / object-level authorization (change the ID in the request — whose data comes back?)
- Cross-tenant / cross-workspace isolation (every query: where is the tenant filter?)
- Missing server-side auth (client-side gating only)
- Injection: SQL, XSS, command, header, and prompt injection via any content an LLM reads
- Secrets: keys in code, tokens in logs, credentials in client bundles
- Unverified webhooks / missing signature or timestamp checks
- Public-share URLs: enumeration, expiry, permission honored after revocation

## 3. Claim-vs-code
Take every specific promise in the README / landing page / docs / FAQ / security page — numbers, guarantees, feature names — and find the code that delivers it. "AES-256 encrypted", "undo within 5 minutes", "we never train on your data", "sub-second search": each is either verified or it's a finding. Fake/staged demo data presented as real counts too.

## 4. Data-shape
Re-run key flows mentally (or actually) at the extremes: zero items, one item, partially-filled records, unicode/emoji/RTL, very long strings, 100k rows. Pagination without limits, unbounded `IN` clauses, UI that assumes at least one element, reducers that crash on empty.

## 5. Platform-divergence
Compare every surface the product ships against the others: web vs mobile vs CLI vs API. Feature present on one and silently absent on another; same action with different validation; dark-mode/responsive breakage; keyboard-only operation.

## 6. Lifecycle
Follow one actor through time: signup → onboarding → daily use → plan change → member leaves → account deletion. Offboarding is where data-retention promises die: does departed-user data actually get purged? Do revoked integrations stop syncing? Does deletion cascade or strand orphans?

## 7. Write-path integrity
Every write with an external effect (email, webhook, payment, third-party API): is it idempotent under retry? What happens if the process dies between the two writes? Look specifically for non-atomic sibling writes — record persisted before its permission row, flag set before the index is built — these create windows where data is live but unprotected, and they are invisible to tests.

## 8. Failure-mode
For each dependency (database, queue, third-party API, LLM provider): what does the product do when it's down, slow, or returns garbage? Missing timeouts, retries without backoff, errors swallowed into silent no-ops, dead-letter handling, malformed-response parsing.

## 9. Dead-and-stale
Hunt code and content that lies about the present: docs for removed features, screenshots of old UI, TODO/FIXME shipping in prod paths, feature flags permanently off with live marketing, exported symbols nothing imports, migrations that drifted from the schema.

## 10. Gate-escape
Find defects that pass the project's own quality gates: type errors excluded from the typecheck path, generated code not covered by CI, lint suppressions hiding real issues, tests that assert nothing, builds configured to ignore errors. A gate that can't fail is a finding.

## 11. Perf
N+1 queries, missing indexes on filtered/sorted columns, queries without LIMIT, payloads serializing entire objects for one field, bundle size, render waterfalls, work in loops that should be batched, LLM/API calls without timeout or cap.

## 12. A11y & UX-jank
Focus order, ARIA on interactive elements, contrast, motion without reduce-motion, skeleton/empty/error states, layout shift, forms that lose input on error, destructive actions without confirm or undo.

## When the codebase has been audited before

Add these lenses for verification loops (after fixes):
- **Regression**: diff-driven — what did the fixes touch, and what shares those paths?
- **Fix-completeness**: for each fixed finding, hunt its siblings. The same mistake is almost never made once.
- **Over-reach**: did any fix change behavior beyond its finding?
