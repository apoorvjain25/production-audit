# Production Audit — single-file prompt

Not using Claude Code? Paste everything below the line into any capable coding agent (Cursor, Windsurf, Copilot, aider, a raw API call) pointed at your codebase. For Claude Code, install the skill instead — see the README.

---

Run a complete production-readiness audit of this codebase/product. Do not stop for confirmation. Do not summarize. The flat list IS the report.

PRINCIPLES
- Trust what the code does, not what it's called. Open the file. Trace the real path: UI → API → data layer → response → render.
- A public claim without implementing code is itself a HIGH finding (README, landing page, docs, security page — every specific promise gets verified).
- One pass is never enough. Audit in repeated passes, each from a DIFFERENT angle, until two consecutive passes surface zero new findings.
- Attack every finding before reporting it: try to refute it against the real code (guard upstream? enforced elsewhere? actually unreachable?). If you can't pin it to file:line or URL+selector, drop it.
- A short list means you didn't look hard enough. Real products carry hundreds of findings.

PROCESS
1. Silently build a complete feature inventory first: every page/route/screen (including admin pages and settings toggles), every CLI command, API endpoint, webhook, integration, background job, plus every claim in the README/docs/landing pages. This is your coverage checklist — audit every item, however small.
2. Discovery loop. Run passes through diverse lenses, appending only NEW findings: subsystem sweep; attack-class (IDOR, cross-tenant isolation, client-only auth, SQL/XSS/prompt injection, exposed secrets, unverified webhooks, share-link enforcement); claim-vs-code; data-shape (zero/one/partial/huge/unicode data); platform divergence (web/mobile/CLI/API, dark mode, responsive); lifecycle (signup → daily use → offboarding → deletion — do retention promises hold?); write-path integrity (idempotency, non-atomic sibling writes — record live before its permission row); failure modes (each dependency down/slow/garbage); dead-and-stale (docs for removed features, TODOs in prod, flags off with live marketing); gate-escape (defects that pass the project's own typecheck/CI/lint); perf (N+1, missing indexes, no-LIMIT queries, unbounded loops, calls without timeouts); a11y/UX (focus, ARIA, contrast, empty/error states, destructive actions without undo).
3. Stop discovery only when two consecutive diverse passes find nothing new.

PER-FEATURE CHECKS
Implemented end-to-end or stubbed; empty/partial/huge data; loading/error/empty states; server-side authorization on every path that reaches the data; external writes idempotent and audited; parity across every surface the product ships.

OUTPUT FORMAT
A single flat list. No preamble, no overview, no closing summary. Each row, 1–2 lines max:

[SEVERITY] [AREA] path/to/file.ts:123 — what is wrong — one-line fix

SEVERITY = CRITICAL (data loss, breach, broken core flow, crash) / HIGH (claimed feature broken or missing, security weakness, silent failure) / MEDIUM (degraded behavior, edge-case failure, real inconsistency) / LOW (minor bug, polish) / IMPROVEMENT (concrete upgrade only — no hedging).
AREA = whatever fits the product: FRONTEND BACKEND API DB SECURITY AUTH DOCS CONTENT UX A11Y PERF RELIABILITY DATA BUILD MOBILE INTEGRATION CONFIG.
Order CRITICAL → HIGH → MEDIUM → LOW → IMPROVEMENT; group by AREA within severity.

HARD RULES
- Every row has file:line or URL+selector. No location ⇒ drop the row.
- No "consider/might/could/potentially". Concrete defects, concrete fixes.
- Don't skip small features or known-broken areas — same depth everywhere, no commentary.
- If you run out of context, end with exactly one line: TRUNCATED AT [area] — [N] inventory items unchecked. Nothing after it.

IF ASKED TO FIX
Fix in severity waves (1: CRITICAL+HIGH, 2: MEDIUM, 3: LOW+IMPROVEMENT), gating each wave on green typecheck + full test suite, then reviewing every change for over-reach (anything changed beyond its finding gets reverted). Then run a verification loop: re-audit with fresh angles and fix regressions, round after round, until two consecutive passes find zero CRITICAL and zero HIGH. Expect 10+ rounds on a real codebase.
