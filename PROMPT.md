# Production Audit: single-file prompt

Not using Claude Code? Paste everything below the line into any capable coding agent (Cursor, Windsurf, Copilot, aider, a raw API call) pointed at your codebase. For Claude Code, install the skill instead. See the README.

---

Run a complete production-readiness audit of this codebase/product. Do not stop for confirmation. Do not summarize. The flat list IS the report.

This is a defensive engagement: the codebase under audit is one the user owns or is authorized to test and modify. Find defects, prove them against the real code, and, when asked to fix, remediate them. Do not produce working exploits, attack live systems, or probe anything outside this codebase.

PRINCIPLES
- Trust what the code does, not what it's called. Open the file. Trace the real path: UI → API → data layer → response → render.
- A public claim without implementing code is itself a HIGH finding (README, landing page, docs, security page: every specific promise gets verified).
- One pass is never enough. Audit in repeated passes, each from a DIFFERENT angle, until two consecutive passes surface zero new findings.
- Attack every finding before reporting it: try to refute it against the real code (guard upstream? enforced elsewhere? actually unreachable?). If you can't pin it to file:line or URL+selector, drop it.
- A short list means you didn't look hard enough. Real products carry hundreds of findings.

PROCESS
1. Silently build a complete feature inventory first: every page/route/screen (including admin pages and settings toggles); every interactive UI element and state (forms + validation, modals, toasts, dropdowns, search/filter/sort, bulk actions, keyboard shortcuts, undo, pagination; loading/empty/error/offline/404/500); auth surfaces (signup, login, password reset, OAuth/SSO, MFA, sessions, invitations, roles, account deletion); every CLI command, API endpoint, webhook, integration, background job; file flows (upload/download, import/export, previews); outbound comms (emails, push, SMS, in-app notifications); billing flows (checkout, trials, upgrades, payment failure, cancellation); public surfaces (share links, embeds, sitemap/robots, OG/meta, deep links); realtime channels; static assets (icons, images, animations, fonts, favicons); LLM prompts and model configs; plus every claim in the README/docs/landing/legal/security pages, across every locale, theme, viewport, and platform shipped. This is your coverage checklist: audit every item, however small.
2. Run the project's own gates before anything else: clean install, full build, typecheck across every package, lint, complete test suite. Every failure or ignored warning is a finding, and so is a gate configured so it cannot fail (ignoreBuildErrors, `|| true`, skipped tests, suppressions).
3. Discovery loop. Each pass sweeps the whole inventory through ONE lens. Never repeat a lens; append only NEW findings (de-duplicate by file+issue) and keep a pass ledger (`pass #, lens, new findings`). The lenses:
   - Subsystem sweep: one subsystem at a time (auth, billing, search, ...), entry points traced to storage and back. Best first pass.
   - Attack-class: IDOR, cross-tenant isolation, client-only auth, SQL/XSS/command/prompt injection, exposed secrets, unverified webhooks, share-link enforcement after revocation.
   - Auth & permissions deep-dive: full role x action matrix enforced server-side; session expiry, rotation, server-side logout invalidation; token signature/expiry validation and revocation; single-use expiring reset tokens; no user enumeration; MFA recovery bypasses; demotion strips access immediately; permission caches never serve stale grants; safe default permissions on new resources.
   - Claim-vs-code: every specific public promise (numbers, guarantees, feature names) traced to the code that delivers it; staged demo data presented as real counts too.
   - Data-shape: zero/one/partial/huge data, unicode/emoji/RTL, very long strings, 100k rows; pagination without limits; UI assuming at least one element.
   - Platform divergence & responsiveness: web vs mobile vs CLI vs API parity; dark mode; every page at phone/tablet/desktop widths (overflow, fixed widths, broken breakpoints, touch targets).
   - Lifecycle: signup → onboarding → daily use → plan change → offboarding → deletion; retention promises actually honored, revoked integrations actually stop, deletion cascades.
   - Write-path integrity: external-effect writes idempotent under retry; non-atomic sibling writes (record live before its permission row).
   - Failure-mode: each dependency down/slow/garbage: timeouts, backoff, swallowed errors, dead-letter paths, malformed-response parsing.
   - Dead-and-stale: docs for removed features, TODO/FIXME in prod paths, flags off with live marketing, old screenshots, drifted migrations.
   - Gate-escape: defects that pass the gates run in step 2 (type errors outside the typecheck path, generated code outside CI, assertion-free tests).
   - Perf: N+1, missing indexes, no-LIMIT queries, unbounded loops, render waterfalls, bundle size; Core Web Vitals, unoptimized images, render-blocking assets, missing cache headers; calls without timeout or cap.
   - A11y & UX-jank: focus order, ARIA, contrast, reduce-motion, skeleton/empty/error states, forms losing input, destructive actions without confirm or undo; animations that stutter, never finish, or block input; spinners with no failure path.
   - Content & copy: typos, placeholder text, jargon on non-technical pages, stack traces/JSON/NaN rendered to users; every number, stat, and price identical across landing/pricing/docs/app.
   - Asset & icon integrity: broken images/icons, mixed icon sets, icons contradicting their action, missing favicon/OG/touch icons, missing alt text, fonts that flash or never load.
   - Connection & wiring: frontend calls to dead/renamed endpoints, hardcoded localhost/staging URLs, secrets in the client bundle, CORS wider than needed; DB pool size vs concurrency, leaked connections, no acquire timeout, missing TLS; test keys in prod, env mismatches between duplicated configs.
   - LLM & prompt quality: prompts contradicting their output parsers, unvalidated model output with no fallback, missing token/cost/timeout caps, deprecated model IDs, near-duplicate prompts drifted apart, PII to providers, user content concatenated where instructions live.
   - Resource leaks & long-running drift: listeners/intervals/observers never cleaned up, unbounded in-memory caches, unclosed file/socket/DB handles, workers growing with uptime, temp files accumulating, logs without rotation.
   - Observability & ops: errors swallowed untracked, unlogged critical paths, PII in logs, no health checks, no graceful shutdown, no alerts on job failures, stack traces or source maps in prod.
   - Abuse & limits: no rate limits on auth/email/expensive endpoints, unbounded uploads and payloads, uncapped LLM spend, missing storage quotas, no bot resistance where it matters.
   - Config & environment: env vars unvalidated at boot, dev defaults in prod, debug mode reachable, drifted duplicate configs, untested flag states, real secrets in committed examples.
   - Dependency & supply-chain: known CVEs in the lockfile, deprecated packages, unpinned versions, duplicate versions of one library, install scripts, license conflicts.
   - Caching correctness: cache keys missing tenant/user scope, stale after writes, authorization cached past revocation, CDN caching authenticated responses, localStorage surviving logout.
   - Concurrency & races: double-submit, two tabs, two workers on one job, webhooks delivered twice, check-then-act without unique constraints, last-write-wins edits, non-atomic counters, job locks that don't expire.
4. Stop discovery only when two consecutive diverse passes find nothing new.
5. If your harness supports subagents or parallel tasks, fan out one finder per lens/subsystem and route every candidate finding through a skeptic agent prompted to refute it. If not, run lenses sequentially yourself with the same skeptic step.

SCOPING
The user may scope the run (one subsystem like src/billing, one lens family like security, or docs-vs-code). Apply the same loop, lenses, and rules to the narrowed inventory. No scope given = the whole product.

PER-FEATURE CHECKS
Implemented end-to-end or stubbed; empty/partial/huge data; loading/error/empty states; server-side authorization on every path that reaches the data, correct for every role (not just the happy path); external writes idempotent and audited; parity across every surface the product ships.

OUTPUT FORMAT
A single flat list. No preamble, no overview, no closing summary. Each row, 1-2 lines max:

[SEVERITY] [AREA] path/to/file.ts:123 - what is wrong - one-line fix

For example:
[CRITICAL] [SECURITY] src/lib/cache.ts:21 - dashboard cache key omits the workspace id; one tenant's data served to another - add the tenant to the key
[HIGH] [CONTENT] landing/security.html §hero - claims "AES-256 encryption at rest"; no encryption configured in the storage layer - implement it or remove the claim
[MEDIUM] [PERF] src/dashboard/page.tsx:61 - members fetched per project in a loop (N+1) - one grouped query

SEVERITY = CRITICAL (data loss, breach, broken core flow, crash) / HIGH (claimed feature broken or missing, security weakness, silent failure) / MEDIUM (degraded behavior, edge-case failure, real inconsistency) / LOW (minor bug, polish) / IMPROVEMENT (concrete upgrade only, no hedging).
AREA = whatever fits the product: FRONTEND BACKEND API DB SECURITY AUTH DOCS CONTENT UX A11Y PERF RELIABILITY DATA BUILD MOBILE INTEGRATION CONFIG AI.
Order CRITICAL → HIGH → MEDIUM → LOW → IMPROVEMENT; group by AREA within severity.

HARD RULES
- Every row has file:line or URL+selector. No location means drop the row.
- No "consider/might/could/potentially". Concrete defects, concrete fixes.
- Don't skip small features or known-broken areas; same depth everywhere, no commentary.
- If you run out of context, end with exactly one line: TRUNCATED AT [area] - [N] inventory items unchecked. Nothing after it.

IF ASKED TO FIX
Fix in severity waves (1: CRITICAL+HIGH, 2: MEDIUM, 3: LOW+IMPROVEMENT), gating each wave on green clean build + typecheck + lint + full test suite, then reviewing every change for over-reach (anything changed beyond its finding gets reverted). Deferrals are listed with a reason, never silently dropped. Then run a verification loop: re-audit with fresh angles (regression, fix-completeness, over-reach; the same mistake is almost never made once) and fix what surfaces, round after round, until two consecutive passes find zero CRITICAL and zero HIGH. Expect 10+ rounds on a real codebase.
