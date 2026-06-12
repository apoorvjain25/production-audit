---
name: production-audit
description: Exhaustive production-readiness audit of any codebase or product. Runs diverse-angle discovery passes until two consecutive passes find nothing new, adversarially verifies every finding against the real code, and reports one flat severity-tagged list with file:line evidence — no summary, no hedging. Use when asked to audit a platform/codebase/product, do a pre-launch review, or "find everything wrong".
---

# Production Audit

You are running a complete production-readiness audit. The goal is not "a list of issues" — it is **everything that is wrong**, proven against the real code, reported in a format a team can fix row by row.

Do NOT stop for confirmation between phases. Do NOT summarize. The flat list IS the report.

## Core principles

1. **Trust what the code does, not what it's called.** A function named `validatePermissions` proves nothing. Open it. Trace the actual path: UI → API → data layer → response → render.
2. **A claim without code is a finding.** If the README, landing page, docs, or marketing copy promises something the code doesn't do, that's a HIGH-severity finding, not an observation.
3. **One pass is never enough.** A single audit pass finds what that pass's framing makes visible. Converge instead: keep auditing from *different angles* until two consecutive passes surface zero new findings.
4. **Every finding gets attacked before it gets reported.** Before a row makes the list, try to refute it against the real code. Plausible-but-wrong findings destroy trust in the whole list.
5. **A short list means you didn't look hard enough.** Real products carry hundreds of findings. Breadth is the deliverable.

## Phase 0 — Feature inventory (silent)

Before auditing anything, build a complete inventory of what the product claims to be and do. Walk, as applicable:

- Marketing/landing pages, feature pages, pricing page, FAQ, manifesto/about
- README, docs site, API reference, changelog
- Every route/page/screen in the app — including admin pages, settings panels, and toggles
- CLI commands and flags, public API endpoints, webhooks, integrations
- Background jobs, crons, workers, queues

Count the inventory and keep it as your coverage checklist. Audit *every* item — a buried settings toggle is as in-scope as the home page. Do not show the inventory to the user; it drives coverage, not output.

## Phase 1 — Discovery loop (front convergence)

Run repeated audit passes. **Each pass takes a different angle** — never repeat a lens. Pick angles from `references/audit-angles.md` (subsystem sweep, attack-class, claim-vs-code, data-shape, platform-divergence, lifecycle, dependency/infra, …) and what to look for from `references/finding-taxonomy.md`.

After each pass:
- Append only NEW findings to the master list (de-duplicate by file+issue).
- **Stop when two consecutive diverse passes surface zero new findings.** One quiet pass is not convergence — the diversity of angles is what makes "we found everything" credible.

For every candidate finding, verify before it makes the list:
- Open the actual file/page. Confirm the issue exists at that exact location, today.
- Try to refute it: is there a guard upstream? Is the dead code actually unreachable? Is the "missing check" enforced elsewhere?
- If you cannot pin it to a `file:line` or a `URL + selector`, drop it.

If your harness supports subagents, fan out parallel finders (one per angle/subsystem) and route each candidate finding through a skeptic agent prompted to refute it. If not, run angles sequentially yourself with the same skeptic step.

### Per-feature checks (apply to every inventory item)

- Implemented end-to-end, or stubbed/half-wired/TODO?
- Behavior with empty data, partial data, and huge data
- Loading, error, and empty states
- Authorization: is the server-side check actually there, on every path that reaches the data?
- Writes with external effects: idempotent? audited/logged? retried safely?
- Parity across surfaces the product ships (web/mobile/CLI/API, light/dark, desktop/responsive)

## Output format

A single flat list. No preamble, no overview, no closing summary. Each row, 1–2 lines max:

```
[SEVERITY] [AREA] path/to/file.ts:123 — what is wrong — one-line fix
```

- **SEVERITY**: `CRITICAL` (data loss, security breach, broken core flow, crash) / `HIGH` (claimed feature broken or missing, security weakness, silent failure) / `MEDIUM` (degraded behavior, edge-case failure, real inconsistency) / `LOW` (minor bug, polish) / `IMPROVEMENT` (concrete upgrade — still no hedging)
- **AREA**: pick what fits the product, e.g. `FRONTEND` `BACKEND` `API` `DB` `SECURITY` `AUTH` `DOCS` `CONTENT` `UX` `A11Y` `PERF` `RELIABILITY` `DATA` `BUILD` `MOBILE` `INTEGRATION` `CONFIG`
- Order CRITICAL → HIGH → MEDIUM → LOW → IMPROVEMENT; within a severity, group by AREA.

### Hard rules

- Every row has `file:line` or `URL + selector`. No location ⇒ the row is dropped.
- No "consider", "might", "could", "potentially". Concrete defects and concrete fixes only.
- If you run out of context, end with exactly one line: `TRUNCATED AT [area] — [N] inventory items unchecked`. Do not pad, do not summarize what you found.
- Do not skip "small" features. Do not skip areas known to be broken — audit them at the same depth, without extra commentary.

## Phase 2 — Fix waves (only when asked to fix)

If the user asked for audit + fix (e.g. `/production-audit fix`):

1. **Wave 1**: CRITICAL + HIGH. **Wave 2**: MEDIUM. **Wave 3**: LOW + IMPROVEMENT.
2. Gate every wave: typecheck + full test suite green before the next wave starts.
3. After each wave, run an **over-reach review**: diff every change against the finding it fixes; revert anything that changed behavior beyond the fix.
4. Handle deferrals explicitly — a finding you won't fix now gets listed as deferred with a reason, never silently dropped.

## Phase 3 — Verification loop (back convergence, only after fixing)

Fixing introduces regressions; the audit isn't done until that's disproven:

1. Re-audit (fresh angles, same rules) + fix anything found, round after round.
2. **Stop only when two consecutive passes find zero CRITICAL and zero HIGH findings.** Expect this to take many passes on a real codebase — 10+ is normal, not failure.
3. Each round's fixes get the same gate (typecheck + tests) and over-reach review.

## Scoping

The user may scope the run: `/production-audit security`, `/production-audit src/billing`, `/production-audit docs-vs-code`. Apply the same loop and rules to the narrowed inventory. Default (no args) = the whole product.
