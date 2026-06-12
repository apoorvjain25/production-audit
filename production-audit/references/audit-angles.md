# Audit angles: the discovery-pass lens catalog

Each discovery pass takes ONE lens from this catalog and sweeps the whole inventory through it. Never run the same lens twice. Convergence ("two consecutive passes find nothing new") only means something if the passes are genuinely diverse: ten subsystem sweeps are one angle, not ten.

Pick the next lens by asking: *which framing has not yet had a chance to make a defect visible?*

Keep a pass ledger as you go: `pass #, lens, new findings` (e.g. `7. failure-mode → 12 new`). Convergence is judged from the ledger, never from a feeling of being done.

The example row under each lens is illustrative, not a template; your rows must point at real locations in the audited product.

## 1. Subsystem sweep
Walk one subsystem at a time (auth, billing, search, notifications, import/export, admin, ...) end to end. Read the entry points, trace to storage and back. Best first pass; it builds the mental map the other lenses need.

> `[CRITICAL] [BACKEND] src/billing/invoice.ts:88 - invoice total computed from client-supplied line-item prices, not server-side prices - recompute from the DB before charging`

## 2. Attack-class
Pick one vulnerability class and hunt it everywhere:
- IDOR / object-level authorization (change the ID in the request: whose data comes back?)
- Cross-tenant / cross-workspace isolation (every query: where is the tenant filter?)
- Missing server-side auth (client-side gating only)
- Injection: SQL, XSS, command, header, and prompt injection via any content an LLM reads
- Secrets: keys in code, tokens in logs, credentials in client bundles
- Unverified webhooks / missing signature or timestamp checks
- Public-share URLs: enumeration, expiry, permission honored after revocation

> `[CRITICAL] [SECURITY] src/api/notes/[id]/route.ts:19 - GET fetches by id with no owner check; any authenticated user can read any note - filter by the session user's workspace`

## 3. Claim-vs-code
Take every specific promise in the README / landing page / docs / FAQ / security page (numbers, guarantees, feature names) and find the code that delivers it. "AES-256 encrypted", "undo within 5 minutes", "we never train on your data", "sub-second search": each is either verified or it's a finding. Fake/staged demo data presented as real counts too.

> `[HIGH] [CONTENT] landing/index.html §features - "unlimited version history" but versions are pruned after 30 days in jobs/prune-versions.ts:12 - fix the claim or the cron`

## 4. Data-shape
Re-run key flows mentally (or actually) at the extremes: zero items, one item, partially-filled records, unicode/emoji/RTL, very long strings, 100k rows. Pagination without limits, unbounded `IN` clauses, UI that assumes at least one element, reducers that crash on empty.

> `[MEDIUM] [FRONTEND] src/components/Chart.tsx:54 - reduce() over data.points with no initial value; crashes on an empty dataset - add an initial value and an empty state`

## 5. Platform-divergence & responsiveness
Compare every surface the product ships against the others: web vs mobile vs CLI vs API. Feature present on one and silently absent on another; same action with different validation; dark-mode breakage; keyboard-only operation. Then sweep every page at phone/tablet/desktop widths: overflow and horizontal scroll, fixed pixel widths, content hidden but still focusable, touch targets too small to hit, breakpoints where the layout breaks instead of adapting.

> `[MEDIUM] [MOBILE] app/screens/Export.tsx - CSV export exists on web, silently absent on mobile - ship it or point users to web`

## 6. Lifecycle
Follow one actor through time: signup → onboarding → daily use → plan change → member leaves → account deletion. Offboarding is where data-retention promises die: does departed-user data actually get purged? Do revoked integrations stop syncing? Does deletion cascade or strand orphans?

> `[HIGH] [DATA] src/jobs/delete-account.ts:40 - deletion removes the user row but leaves their uploads in object storage forever - cascade the delete to S3`

## 7. Write-path integrity
Every write with an external effect (email, webhook, payment, third-party API): is it idempotent under retry? What happens if the process dies between the two writes? Look specifically for non-atomic sibling writes (record persisted before its permission row, flag set before the index is built): these create windows where data is live but unprotected, and they are invisible to tests.

> `[HIGH] [DATA] src/import/processor.ts:142 - document row committed before its permission row, non-atomically; content is live and unprotected in the gap - one transaction, permission first`

## 8. Failure-mode
For each dependency (database, queue, third-party API, LLM provider): what does the product do when it's down, slow, or returns garbage? Missing timeouts, retries without backoff, errors swallowed into silent no-ops, dead-letter handling, malformed-response parsing.

> `[HIGH] [RELIABILITY] src/lib/llm.ts:27 - provider call has no timeout; one hung request pins a worker forever - add a timeout and abort handling`

## 9. Dead-and-stale
Hunt code and content that lies about the present: docs for removed features, screenshots of old UI, TODO/FIXME shipping in prod paths, feature flags permanently off with live marketing, exported symbols nothing imports, migrations that drifted from the schema.

> `[MEDIUM] [DOCS] docs/integrations.md:9 - documents a Slack integration removed three releases ago - delete the section`

## 10. Gate-run & gate-escape
First, actually run the project's own gates: clean install, full build, typecheck across every package/tsconfig, lint, complete test suite. Every error, and every warning the team has learned to ignore, is a finding. Then find defects that pass the gates: type errors excluded from the typecheck path, generated code not covered by CI, lint suppressions hiding real issues, tests that assert nothing or are skipped, builds configured to ignore errors (`ignoreBuildErrors`, `|| true`). A gate that can't fail is a finding.

> `[HIGH] [BUILD] next.config.js:11 - typescript.ignoreBuildErrors: true; 23 type errors ship silently - fix the errors, remove the flag`

## 11. Perf
N+1 queries, missing indexes on filtered/sorted columns, queries without LIMIT, payloads serializing entire objects for one field, bundle size, render waterfalls, work in loops that should be batched, LLM/API calls without timeout or cap. On the web side: Core Web Vitals (LCP, CLS, INP), unsized/unoptimized/eagerly-loaded images, render-blocking scripts and fonts, missing cache headers and compression, animations that thrash layout.

> `[MEDIUM] [PERF] src/dashboard/page.tsx:61 - members fetched per project in a loop (N+1, ~40 queries per load) - one grouped query`

## 12. A11y & UX-jank
Focus order, ARIA on interactive elements, contrast, motion without reduce-motion, skeleton/empty/error states, layout shift, forms that lose input on error, destructive actions without confirm or undo. Animation quality: transitions that stutter, never finish, or block input; spinners that spin forever instead of surfacing the error; an animation present on one surface and missing on its sibling.

> `[MEDIUM] [A11Y] src/components/Modal.tsx - focus not trapped; Tab escapes into the page behind the overlay - trap focus, restore it on close`

## 13. Content & copy
Read every page as a first-time, non-technical user. Typos and grammar; placeholder text (lorem ipsum, "test", TODO) shipped to prod; jargon where plain language belongs; raw technical artifacts rendered as content: stack traces, error codes, JSON blobs, `NaN`/`undefined`/`null` on screen. Terminology consistent across pages, and every number, stat, and price identical everywhere it appears: landing vs pricing vs docs vs in-app.

> `[LOW] [CONTENT] pricing.html §faq - "recieve" twice, and the failed-payment state renders raw "Error: ECONNREFUSED" - fix the copy, map errors to human text`

## 14. Asset & icon integrity
Broken or missing images and icons (404'd files, empty boxes, fallback glyphs); mixed icon sets on one surface; an icon whose meaning contradicts its action; missing favicon, OG/social-preview image, and touch icons; images without alt text; fonts that flash, swap late, or never load.

> `[LOW] [UX] public/ - no favicon or touch icons; tabs and home-screen pins show the browser default - add the icon set`

## 15. Connection & wiring
Audit every connection the product makes. Frontend → API: calls to endpoints that don't exist or were renamed, hardcoded localhost/staging URLs, secrets or env vars leaking into the client bundle, CORS wider than needed, fetches with no error handling. App → database: pool size vs real concurrency, connections acquired and never released, no acquire timeout, missing TLS, a new connection per request under serverless. App → third parties: test/sandbox keys in prod, region or environment mismatches between duplicated configs.

> `[CRITICAL] [CONFIG] src/lib/api.ts:4 - API base URL hardcoded to the staging host; production builds call staging - read from env, validate at boot`

## 16. LLM & prompt quality
Only if the product ships LLM features. Then read every prompt the way the model will. Ambiguous or self-contradicting instructions; prompt and output parser disagreeing on format (parser expects JSON the prompt never demands); model output flowing into code paths unvalidated, with no fallback for garbage; missing token, cost, and timeout caps; deprecated or hardcoded model IDs; near-duplicate prompts that drifted apart; user PII forwarded to the provider without need; user content concatenated into the instruction segment (injection-prone assembly; overlaps attack-class, hunt it per prompt here).

> `[HIGH] [AI] src/prompts/extract.ts:18 - prompt asks for prose but the parser JSON.parses the reply; the catch block returns [] so every extraction silently yields nothing - demand JSON in the prompt, validate, surface failures`

## 17. Auth & permissions deep-dive
The attack-class lens samples; this lens is exhaustive. Build the full role x action matrix (anonymous, member, admin, every custom role x every route, mutation, export, and admin surface) and verify each cell is enforced server-side. Sessions: expiry, rotation on login and privilege change, server-side invalidation on logout and password change, concurrent-session handling. Tokens: signature/algorithm/expiry actually validated, refresh rotation, revocation honored. Password reset: tokens single-use and expiring, no user enumeration via login/reset/invite responses. MFA: enrollment, recovery codes, and whether any path skips it. Role changes: demotion strips access immediately; removed members and revoked invites lose everything; permission caches don't serve stale grants. New resources get safe default permissions.

> `[CRITICAL] [AUTH] src/api/admin/users.ts:9 - admin role checked only in the UI; the endpoint returns every user to any authenticated session - enforce the role server-side`

## 18. Resource leaks & long-running drift
What grows with uptime? Memory: event listeners, intervals, and observers registered and never cleaned up (effect hooks without cleanup); unbounded in-memory caches and maps; collections that only ever append; closures pinning large objects. Handles: files, streams, sockets, and DB cursors opened and never closed. Processes: workers, websocket servers, and crons that degrade over days; temp files that accumulate; logs without rotation filling disk. Anything fine in a 5-minute test and broken after a week is this lens's quarry.

> `[HIGH] [RELIABILITY] src/realtime/hub.ts:33 - per-connection event handlers never unsubscribed on disconnect; hub memory grows with every connect/disconnect cycle - clean up in the close handler`

## 19. Observability & operations
Could the team even tell this product is broken? Errors caught and dropped with no tracking; critical paths (payments, auth, deletes) with zero logging; PII or secrets written to logs; no health/readiness endpoints; no alerting on job failures or queue depth; no graceful shutdown (in-flight requests and jobs dropped on every deploy); debug endpoints, verbose stack-trace errors, or source maps exposed in prod.

> `[HIGH] [RELIABILITY] src/api/payments/webhook.ts:51 - catch (e) {} around payment confirmation; failed confirmations are invisible - log, track, alert`

## 20. Abuse & limits
What can a hostile or careless user do unboundedly? Auth endpoints (login, signup, reset) without rate limits; user-triggered email (invites, verification resend) usable as a spam cannon; uploads without size/type caps; request payloads without limits; expensive operations (LLM calls, exports, reports) callable in a loop with no quota or cost cap; missing storage quotas; signup without bot resistance where it matters.

> `[HIGH] [SECURITY] src/api/auth/reset/route.ts - no rate limit on password-reset email; one loop floods a victim's inbox and burns sender reputation - rate-limit per account and per IP`

## 21. Config & environment
Required env vars read lazily and unvalidated, crashing mid-request instead of refusing to boot; dev defaults (localhost, test keys, weak secrets) that survive into prod; debug mode reachable in prod; the same setting duplicated across configs and drifted; feature flags where only one state was ever tested; real secrets in `.env.example` or committed config files.

> `[HIGH] [CONFIG] src/lib/stripe.ts:3 - falls back to a sk_test_... key when STRIPE_KEY is unset; prod silently takes test payments - fail at boot when it's missing`

## 22. Dependency & supply-chain
Known CVEs in the lockfile (run the ecosystem's audit tool); deprecated or abandoned packages on critical paths; unpinned versions or lockfile drift from the manifest; the same library at multiple versions; install scripts doing more than installing; licenses incompatible with how the product ships.

> `[HIGH] [BUILD] package.json:21 - jsonwebtoken pinned at 8.5.1 (CVE-2022-23529, verification bypass) - upgrade to 9.x`

## 23. Caching correctness
For every cache: scoped right, invalidated when? Cache keys missing the tenant/user dimension, serving one user's data to another; authorization decisions cached past revocation; stale reads after writes with no invalidation path; CDN/HTTP caching of authenticated responses (missing `Cache-Control: private`); client-side storage (localStorage, service workers) surviving logout or schema changes.

> `[CRITICAL] [SECURITY] src/lib/cache.ts:21 - dashboard cache key is the route only, no workspace id; one tenant's dashboard is served to the next - add the tenant to the key`

## 24. Concurrency & races
Two of anything at once: a user double-clicking submit; the same form open in two tabs; two workers grabbing the same job; a webhook delivered twice. Check-then-act gaps with no unique constraint as the last line of defense; concurrent edits where the last write silently wins; counters updated read-modify-write instead of atomically; job locks that don't expire when a worker dies.

> `[HIGH] [DATA] src/api/redeem/route.ts:30 - coupon checked then decremented in two steps; parallel requests over-redeem - atomic decrement guarded by a constraint`

## When the codebase has been audited before

Add these lenses for verification loops (after fixes):
- **Regression**: diff-driven. What did the fixes touch, and what shares those paths?
- **Fix-completeness**: for each fixed finding, hunt its siblings. The same mistake is almost never made once.
- **Over-reach**: did any fix change behavior beyond its finding?
