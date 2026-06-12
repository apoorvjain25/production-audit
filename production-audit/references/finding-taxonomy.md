# Finding taxonomy — what counts as a finding

Use this as the checklist of defect classes. Anything here, pinned to a `file:line` or `URL + selector` and surviving a refutation attempt, is a row.

## Bugs
Dead links and 404s; crashes and unhandled rejections; silent failures (caught-and-dropped errors); race conditions; timeouts missing or mishandled; off-by-one and boundary errors; state that survives logout; double-submit side effects.

## Incomplete
TODO/FIXME in shipping paths; stubbed functions returning fixtures; hardcoded values that should be config; half-wired features (UI exists, handler doesn't — or the reverse); marketing claims with no implementing code; settings that render but change nothing.

## Security
IDOR and missing object-level authorization; cross-tenant data leakage; auth checks only on the client; SQL/XSS/command injection; prompt injection through any content an LLM ingests (retrieved documents, user fields, web content); exposed secrets and tokens in code, logs, or client bundles; unverified webhook signatures; missing replay protection; permission checks bypassed by alternate routes (export, search, share, API) to the same data; share-link enforcement after revocation; audit-log gaps on sensitive operations.

## Stale content
Docs for features that no longer exist; shipped features with no docs; outdated claims and numbers; screenshots of old UI; changelogs that stopped; demo/sample data presented as real.

## Inconsistency
Same concept named differently across surfaces; UI contradicting itself (count says 3, list shows 2); validation rules differing between client and server; divergence between platforms (web vs mobile vs API vs CLI); dark-mode/responsive breakage.

## UX & A11y
Missing loading/empty/error states; layout shift; forms losing input on error; destructive actions without confirm or undo; focus traps and broken tab order; missing ARIA on interactive elements; insufficient contrast; animation without reduce-motion support; touch targets too small.

## Performance
N+1 queries; missing indexes on filtered/sorted/joined columns; queries without LIMIT; unbounded loops over user-controlled collections; oversized payloads and bundles; sequential awaits that should be parallel; LLM/third-party calls with no timeout, retry cap, or cost cap.

## Data integrity
Missing foreign keys or constraints the code assumes; soft-delete honored in some queries and not others; non-atomic multi-step writes (record live before its permission/index/flag sibling); migration drift from the live schema; orphaned records on delete; missing idempotency keys on retried writes; timezone/encoding mishandling.

## Reliability
No retry or backoff on transient failures; no dead-letter path for failed jobs; webhook delivery without retry; behavior when a dependency is down (database, queue, LLM, third-party API); malformed external responses crashing parsers; crons that overlap themselves; queues with no visibility into depth or failure.

## Promises-vs-reality
Verify EVERY specific public claim against code: encryption standards named on the security page; retention and deletion guarantees; "we never X" statements; quantified claims (accuracy %, latency, limits); compliance statements; pricing-page feature matrices. Each unverifiable or false claim is a finding — usually HIGH.

## Severity guide

- **CRITICAL** — data loss or corruption; security breach reachable today; core flow broken for all users; crash on a primary path.
- **HIGH** — claimed feature broken or absent; security weakness needing one precondition; silent failure of something users rely on; data leak with limited scope.
- **MEDIUM** — degraded or inconsistent behavior; edge-case failure with a workaround; real but bounded perf problem.
- **LOW** — minor bug, cosmetic defect, polish.
- **IMPROVEMENT** — a concrete, specific upgrade (named change at a named location). Not "consider improving X".
