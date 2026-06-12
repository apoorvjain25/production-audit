# Sample report: excerpt from a real run

30 rows of the 1,247 findings from the pre-launch audit of [Pulse](https://pulsehq.tech), a company brain built for the agent era. **Anonymized**: file paths, identifiers, and specifics have been rewritten; the severities, defect classes, and shape of each finding mirror the real rows. The full report is internal; this excerpt shows what the output of a converged run looks like.

## The rows

```
[CRITICAL] [SECURITY] src/lib/cache.ts:21 - workspace dashboard cache key omits the workspace id; one tenant's dashboard served to the next - add the workspace to the key
[CRITICAL] [SECURITY] src/api/documents/[id]/route.ts:84 - share-link content still served after revocation; revokedAt never checked - add revokedAt IS NULL to the access query
[CRITICAL] [SECURITY] src/api/export/route.ts:33 - export path skips the document-level permission filter that search applies; any member can export restricted docs - reuse the same authorization filter on every path to the data
[CRITICAL] [AUTH] src/api/admin/members.ts:9 - admin role checked only in the UI; the endpoint returns every member to any authenticated session - enforce the role server-side
[CRITICAL] [DATA] src/jobs/reindex.ts:57 - reindex deletes search entries before rebuilding with no resume marker; a crash mid-job leaves documents present in every count and absent from every search - rebuild into a shadow index, swap atomically
[HIGH] [DATA] src/import/processor.ts:142 - document row committed before its permission row, non-atomically; content is live and unprotected in the gap - one transaction, permission first
[HIGH] [AI] src/prompts/answer.ts:18 - retrieved document content concatenated into the instruction segment; a document containing instructions can steer the assistant - separate retrieved content into a delimited data block
[HIGH] [AI] src/prompts/extract.ts:31 - prompt asks for prose but the parser JSON.parses the reply; the catch block returns [] so every extraction silently yields nothing - demand JSON, validate, surface failures
[HIGH] [SECURITY] src/api/webhooks/slack.ts:12 - signature verified but timestamp ignored; captured requests are replayable - reject requests older than 5 minutes
[HIGH] [SECURITY] src/api/auth/reset/route.ts:22 - no rate limit on password-reset email, and the response reveals whether the address exists - rate-limit per account and IP, return a uniform response
[HIGH] [AUTH] src/lib/session.ts:74 - password change does not invalidate the user's other sessions - revoke all sessions except the current one on change
[HIGH] [RELIABILITY] src/jobs/webhook-sender.ts:51 - failed deliveries dropped with no retry or dead-letter path - backoff retry + DLQ
[HIGH] [RELIABILITY] src/realtime/hub.ts:33 - per-connection event handlers never unsubscribed on disconnect; hub memory grows with every connect cycle - clean up in the close handler
[HIGH] [CONFIG] src/lib/llm.ts:8 - falls back to a hardcoded provider key when the env var is unset; prod silently bills the wrong account - fail at boot when missing
[HIGH] [CONTENT] landing/security.html §hero - claims "AES-256 encryption at rest"; no encryption configured anywhere in the storage layer - implement it or remove the claim
[MEDIUM] [PERF] src/app/workspace/page.tsx:61 - member counts fetched per project in a loop (N+1, ~40 queries per load) - one grouped query
[MEDIUM] [PERF] src/api/search/route.ts:29 - search query has no LIMIT; a 50k-document workspace serializes the entire result set - paginate with a hard cap
[MEDIUM] [DATA] src/api/integrations/disconnect.ts:40 - disconnect revokes the token but the sync cron keeps running on cached credentials until expiry - stop the sync job in the same transaction
[MEDIUM] [INTEGRATION] src/sync/notion.ts:112 - pagination cursor ignored after the first page; workspaces over 100 pages sync silently partial - loop the cursor to exhaustion
[MEDIUM] [FRONTEND] src/components/Timeline.tsx:54 - reduce() over events with no initial value; crashes on a brand-new workspace - initial value + empty state
[MEDIUM] [UX] src/components/UploadModal.tsx:88 - failed upload clears the file list with no error state; a 413 leaves the spinner running forever - keep the list, show the error, time the spinner out
[MEDIUM] [CONTENT] landing/pricing.html §plans vs in-app billing page - seat limit reads "25" on the landing page and "20" in the app - one source of truth for plan limits
[MEDIUM] [A11Y] src/components/CommandPalette.tsx - no focus trap, and arrow-key navigation lands on items in hidden result groups - trap focus, skip hidden items
[LOW] [CONTENT] landing/faq.html §3 - "recieve" twice - fix the copy
[LOW] [UX] public/ - favicon present but no touch/OG icons; home-screen pins show the browser default - complete the icon set
[LOW] [DOCS] docs/api.md:203 - documents a /v1/summarize endpoint renamed two releases ago - update or redirect
[LOW] [BUILD] tsconfig.json:19 - scripts/ excluded from the typecheck path; 3 type errors live there unseen - include it or typecheck it separately
[IMPROVEMENT] [DATA] src/api/redeem/route.ts:30 - replace the check-then-decrement on coupon redemption with an atomic decrement guarded by a constraint
[IMPROVEMENT] [PERF] src/app/layout.tsx:24 - org, user, and flags fetched with three sequential awaits - Promise.all them
[IMPROVEMENT] [RELIABILITY] src/lib/retry.ts:11 - retries are immediate and unjittered; add exponential backoff with jitter
```

## The discovery ledger

One pass per lens, never repeated. Discovery is allowed to stop only after two consecutive passes find nothing new.

```
 1. subsystem sweep            → 412 new
 2. attack-class               → 167 new
 3. claim-vs-code              → 138 new
 4. gate-run & gate-escape     →  96 new
 5. auth & permissions         →  88 new
 6. data-shape                 →  74 new
 7. lifecycle                  →  61 new
 8. write-path integrity       →  52 new
 9. failure-mode               →  41 new
10. content & copy             →  33 new
11. perf                       →  27 new
12. connection & wiring        →  19 new
13. a11y & UX-jank             →  14 new
14. caching correctness        →   9 new
15. dead-and-stale             →   7 new
16. observability & operations →   5 new
17. concurrency & races        →   3 new
18. abuse & limits             →   1 new
19. config & environment       →   0 new
20. dependency & supply-chain  →   0 new   converged: 1,247 total
```

## The verification ledger (after fix waves)

Re-audit with fresh angles, fix what surfaces, repeat. Done means two consecutive rounds with zero CRITICAL and zero HIGH.

```
round  1 → 31 CRIT/HIGH    round  8 → 3
round  2 → 19              round  9 → 2
round  3 → 12              round 10 → 2
round  4 →  9              round 11 → 1
round  5 →  7              round 12 → 1
round  6 →  5              round 13 → 0
round  7 →  4              round 14 → 0   converged
```

Fourteen rounds is not the skill failing; it's what "done" costs when you require proof. Run it on your own product: `/production-audit`.
