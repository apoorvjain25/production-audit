# Finding taxonomy: what counts as a finding

Use this as the checklist of defect classes. Anything here, pinned to a `file:line` or `URL + selector` and surviving a refutation attempt, is a row. The classes overlap on purpose: a cross-tenant cache key is both Security and Data integrity; file it once, under the class that names the harm.

## Bugs
Dead links and 404s; crashes and unhandled rejections; silent failures (caught-and-dropped errors); race conditions; timeouts missing or mishandled; off-by-one and boundary errors; state that survives logout; double-submit side effects.

## Incomplete
TODO/FIXME in shipping paths; stubbed functions returning fixtures; hardcoded values that should be config; half-wired features (UI exists, handler doesn't, or the reverse); marketing claims with no implementing code; settings that render but change nothing.

## Security
IDOR and missing object-level authorization; cross-tenant data leakage; auth checks only on the client; SQL/XSS/command injection; prompt injection through any content an LLM ingests (retrieved documents, user fields, web content); exposed secrets and tokens in code, logs, or client bundles; unverified webhook signatures; missing replay protection; permission checks bypassed by alternate routes (export, search, share, API) to the same data; share-link enforcement after revocation; audit-log gaps on sensitive operations; sessions not invalidated server-side on logout or password change; reset tokens reusable or non-expiring; user enumeration via login/reset/invite responses; access retained after role demotion or member removal; authorization results cached past revocation; cache keys missing tenant/user scope serving one user's data to another.

## Build & gates
Clean build broken; type errors present or excluded from the typecheck path; lint errors or blanket suppressions; failing, skipped, or assertion-free tests; warnings everyone has learned to ignore; builds or CI configured so errors cannot fail them; generated code outside gate coverage.

## Stale content
Docs for features that no longer exist; shipped features with no docs; outdated claims and numbers; screenshots of old UI; changelogs that stopped; demo/sample data presented as real.

## Content quality
Typos and grammar errors on user-facing pages; placeholder text (lorem ipsum, "test", TODO) in prod; jargon on pages written for non-technical readers; raw technical artifacts rendered to users (stack traces, error codes, JSON blobs, NaN/undefined/null on screen); broken or missing images and icons; mixed icon sets on one surface; an icon contradicting the action it triggers; missing favicon/OG/touch icons; missing or wrong alt text.

## Inconsistency
Same concept named differently across surfaces; UI contradicting itself (count says 3, list shows 2); the same number, stat, price, or limit differing across landing, pricing, docs, and app; validation rules differing between client and server; divergence between platforms (web vs mobile vs API vs CLI); dark-mode/responsive breakage.

## UX & A11y
Missing loading/empty/error states; layout shift; forms losing input on error; destructive actions without confirm or undo; focus traps and broken tab order; missing ARIA on interactive elements; insufficient contrast; animation without reduce-motion support; animations that stutter, never resolve, or block input; spinners with no failure path; touch targets too small.

## Performance
N+1 queries; missing indexes on filtered/sorted/joined columns; queries without LIMIT; unbounded loops over user-controlled collections; oversized payloads and bundles; unsized/unoptimized images without lazy loading; render-blocking scripts and fonts; missing cache headers or compression; sequential awaits that should be parallel; LLM/third-party calls with no timeout, retry cap, or cost cap.

## Data integrity
Missing foreign keys or constraints the code assumes; soft-delete honored in some queries and not others; non-atomic multi-step writes (record live before its permission/index/flag sibling); migration drift from the live schema; orphaned records on delete; missing idempotency keys on retried writes; timezone/encoding mishandling; stale caches after writes with no invalidation path; check-then-act gaps without a unique-constraint backstop; concurrent edits where the last write silently wins; counters updated read-modify-write instead of atomically.

## Reliability
No retry or backoff on transient failures; no dead-letter path for failed jobs; webhook delivery without retry; behavior when a dependency is down (database, queue, LLM, third-party API); malformed external responses crashing parsers; crons that overlap themselves; queues with no visibility into depth or failure; database connection pools exhausted or leaked (clients acquired and never released, no acquire timeout); frontend calling endpoints that don't exist or were renamed; hardcoded localhost/staging URLs in prod builds; test/sandbox keys in prod.

## Resource leaks
Event listeners, intervals, and observers never cleaned up; unbounded in-memory caches and append-only collections; file/stream/socket/DB-cursor handles never closed; workers and long-running processes that grow with uptime; temp files accumulating; logs without rotation filling disk.

## Operations & observability
Errors swallowed with no tracking; critical paths (payments, auth, deletes) with zero logging; PII or secrets in logs; missing health/readiness checks; no graceful shutdown (in-flight work lost on deploy); no alerting on job failures or queue depth; debug endpoints, stack traces, or source maps exposed in prod.

## Abuse & limits
Auth and email-sending endpoints without rate limits; uploads without size/type caps; request payloads without limits; expensive operations (LLM calls, exports, reports) callable in loops with no quota or cost cap; missing storage quotas; signup without bot resistance where it matters.

## Config & environment
Required env vars unvalidated at startup (crash mid-request instead of at boot); dev defaults (localhost, test keys, weak secrets) shipping to prod; debug mode reachable in prod; duplicated-and-drifted config across environments; feature-flag states never tested; real secrets in example or committed config files.

## Dependencies
Known CVEs in locked versions; deprecated or abandoned packages on critical paths; unpinned versions and lockfile drift; the same library at multiple versions; install scripts doing more than installing; licenses incompatible with how the product ships.

## LLM & AI quality
Prompts with ambiguous or conflicting instructions; prompt and output parser disagreeing on format; unvalidated model output entering code paths; no fallback when the model returns malformed content; LLM calls without token, cost, or timeout caps; deprecated or hardcoded model IDs; near-duplicate prompts drifted out of sync; PII sent to the provider without need; user content concatenated into the instruction segment of a prompt.

## Promises-vs-reality
Verify EVERY specific public claim against code: encryption standards named on the security page; retention and deletion guarantees; "we never X" statements; quantified claims (accuracy %, latency, limits); compliance statements; pricing-page feature matrices. Each unverifiable or false claim is a finding, usually HIGH.

## Severity guide

- **CRITICAL**: data loss or corruption; security breach reachable today; core flow broken for all users; crash on a primary path.
  *e.g. any authenticated user can read any other tenant's documents by changing an id*
- **HIGH**: claimed feature broken or absent; security weakness needing one precondition; silent failure of something users rely on; data leak with limited scope.
  *e.g. payment-webhook failures swallowed by an empty catch block; confirmations silently lost*
- **MEDIUM**: degraded or inconsistent behavior; edge-case failure with a workaround; real but bounded perf problem.
  *e.g. dashboard chart crashes when a project has zero data points*
- **LOW**: minor bug, cosmetic defect, polish.
  *e.g. missing favicon; "recieve" on the pricing page*
- **IMPROVEMENT**: a concrete, specific upgrade (named change at a named location). Not "consider improving X".
  *e.g. replace the check-then-decrement on coupon redemption with an atomic decrement*

## Borderline calls

- A security weakness needing one precondition (an authenticated account, a leaked link) is HIGH, not CRITICAL. Reachable today by anyone is what makes CRITICAL.
- A claimed feature failing on common input is HIGH; failing only on exotic input is MEDIUM.
- Anything you would have to hedge to state is not a finding yet. Go verify until the hedge disappears, or drop it.
- Torn between two severities? Take the lower one and make the row's evidence undeniable. An inflated list reads as alarmism and gets ignored; an understated row with perfect evidence still gets fixed.
