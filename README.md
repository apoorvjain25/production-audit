<div align="center">

<img src="assets/hero.png" alt="production-audit: the Claude Code skill that audits your product until it stops finding things, then proves it" width="100%">

# 🔍 production-audit

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Claude Code Skill](https://img.shields.io/badge/Claude%20Code-skill-d97757.svg)](https://docs.claude.com/en/docs/claude-code)
[![Audit lenses](https://img.shields.io/badge/audit%20lenses-24-blue.svg)](production-audit/references/audit-angles.md)
[![Defect classes](https://img.shields.io/badge/defect%20classes-18-blueviolet.svg)](production-audit/references/finding-taxonomy.md)
[![Works everywhere](https://img.shields.io/badge/also%20runs%20in-Cursor%20·%20Windsurf%20·%20Copilot%20·%20aider-8A2BE2.svg)](PROMPT.md)

```
/production-audit
```

</div>

Most "audit my code" prompts do one pass, find 15 issues, and write you a reassuring summary. This skill found **1,200+ real findings** on a production B2B SaaS, then took **14 verification passes** after the fixes before it could honestly say "done."

## How it works

```mermaid
flowchart TD
    P0["Phase 0: Inventory<br/>every page, claim, endpoint, job,<br/>prompt, icon, email, locale"]
    P0 --> LENS["Pick a lens never used before<br/>(24 in the catalog)"]
    LENS --> SWEEP["Sweep the whole inventory through it"]
    SWEEP --> ATTACK["Attack every candidate finding:<br/>try to refute it against the real code"]
    ATTACK --> Q1{"Two consecutive passes<br/>with nothing new?"}
    Q1 -- "no" --> LENS
    Q1 -- "yes" --> REPORT["The report: one flat list<br/>[SEVERITY] [AREA] file:line - defect - fix"]
    REPORT -. "fix mode" .-> WAVE["Fix waves<br/>CRITICAL+HIGH → MEDIUM → LOW"]
    WAVE --> GATE["Gate: build + typecheck + lint + tests<br/>then over-reach review"]
    GATE --> REAUDIT["Re-audit with fresh lenses<br/>(regression, fix-completeness, over-reach)"]
    REAUDIT --> Q2{"Two consecutive passes with<br/>zero CRITICAL / zero HIGH?"}
    Q2 -- "no: fix and go again" --> WAVE
    Q2 -- "yes" --> DONE["Converged. Done, with proof."]
```

The skill inventories every surface your product has (pages, routes, claims, jobs, prompts, icons, emails, locales), then sweeps that inventory through **24 different audit lenses**, one lens per pass. Discovery stops only when **two consecutive passes find nothing new**. In fix mode, a second loop runs after the fixes until **two consecutive passes find zero CRITICAL/HIGH**. "We checked" becomes "we converged."

## Why convergence beats a checklist

|  | A one-shot "audit my code" prompt | `production-audit` |
|---|---|---|
| **Passes** | one | as many as it takes; stops after 2 consecutive quiet passes |
| **Framing** | whatever the prompt happens to emphasize | 24 deliberately diverse lenses, never repeated |
| **False positives** | reported confidently | every finding attacked against the real code before it's reported |
| **Marketing claims** | ignored | every specific public promise verified or flagged |
| **Output** | summary + a few highlights | flat list, every row pinned to `file:line` |
| **"Done" means** | the context window filled up | convergence, proven twice over |

## What a finding looks like

No executive summary. No "overall, the codebase is in good shape." Every row is pinned and actionable:

```
[CRITICAL] [SECURITY] src/lib/cache.ts:21 - dashboard cache key omits the workspace id; one tenant's data served to another - add the tenant to the key
[CRITICAL] [AUTH] src/api/admin/users.ts:9 - admin role checked only in the UI; the endpoint returns every user to any session - enforce the role server-side
[HIGH] [DATA] src/import/processor.ts:142 - document row committed before its permission row, non-atomically; content is live and unprotected in the gap - one transaction, permission first
[HIGH] [CONTENT] landing/security.html §hero - claims "AES-256 encryption at rest"; no encryption configured anywhere in the storage layer - implement it or remove the claim
[HIGH] [AI] src/prompts/extract.ts:18 - prompt asks for prose but the parser JSON.parses the reply; every extraction silently yields [] - demand JSON, validate, surface failures
[HIGH] [RELIABILITY] src/realtime/hub.ts:33 - per-connection handlers never unsubscribed on disconnect; memory grows with every connect cycle - clean up in the close handler
[MEDIUM] [PERF] src/dashboard/page.tsx:61 - members fetched per project in a loop (N+1, ~40 queries per load) - one grouped query
[LOW] [CONTENT] pricing.html §faq - "recieve" twice; failed-payment state renders raw "Error: ECONNREFUSED" - fix the copy, map errors to human text
```

Rules the skill enforces on itself: every row has `file:line` or `URL + selector` (no location means dropped), no "consider/might/could", no padding, and an honest `TRUNCATED AT ...` line if it runs out of context instead of a fake wrap-up.

Want more? See the **[sample report](examples/sample-report.md)**: 30 anonymized rows from the real 1,200-finding run, plus both convergence ledgers.

## The 24 lenses

Each discovery pass takes exactly one lens and sweeps the entire inventory through it; the diversity is what makes "we found everything" credible. Full catalog with a real example finding per lens in [audit-angles.md](production-audit/references/audit-angles.md).

| # | Lens | What it makes visible |
|---|------|----------------------|
| 1 | Subsystem sweep | one subsystem traced end to end; builds the map the other lenses need |
| 2 | Attack-class | IDOR, cross-tenant leaks, injection, exposed secrets, unverified webhooks |
| 3 | Claim-vs-code | every public promise traced to the code that delivers it |
| 4 | Data-shape | zero / one / huge / unicode / 100k-row data through every flow |
| 5 | Platform divergence & responsiveness | web vs mobile vs CLI vs API parity; every page at every width |
| 6 | Lifecycle | signup → daily use → offboarding → deletion; do retention promises hold? |
| 7 | Write-path integrity | idempotency; non-atomic sibling writes (record live before its permission row) |
| 8 | Failure-mode | every dependency down, slow, or returning garbage |
| 9 | Dead-and-stale | docs for removed features, shipping TODOs, flags off with live marketing |
| 10 | Gate-run & gate-escape | actually run build/typecheck/lint/tests, then hunt what slips past them |
| 11 | Perf | N+1, missing indexes, Core Web Vitals, unoptimized images, uncapped calls |
| 12 | A11y & UX-jank | focus, ARIA, contrast, broken animations, spinners with no failure path |
| 13 | Content & copy | typos, placeholder text, jargon, stack traces rendered to users |
| 14 | Asset & icon integrity | broken images, mixed icon sets, missing favicons, fonts that never load |
| 15 | Connection & wiring | dead endpoints, hardcoded staging URLs, DB pool leaks, test keys in prod |
| 16 | LLM & prompt quality | prompts contradicting their parsers, unvalidated output, uncapped spend |
| 17 | Auth & permissions deep-dive | the full role x action matrix, sessions, tokens, resets, MFA, stale grants |
| 18 | Resource leaks & long-running drift | what grows with uptime: listeners, caches, handles, temp files |
| 19 | Observability & operations | could the team even tell it's broken? swallowed errors, no alerts, no logs |
| 20 | Abuse & limits | what a hostile user can do unboundedly: rate limits, quotas, spam vectors |
| 21 | Config & environment | env vars unvalidated at boot, dev defaults in prod, drifted configs |
| 22 | Dependency & supply-chain | CVEs in the lockfile, abandoned packages, license conflicts |
| 23 | Caching correctness | keys missing tenant scope, stale after writes, auth cached past revocation |
| 24 | Concurrency & races | double-submit, two tabs, two workers on one job, check-then-act gaps |

Plus three verification-only lenses for after the fixes: **regression**, **fix-completeness** (the same mistake is almost never made once), and **over-reach**.

## The kinds of bugs it catches that tests don't

From real runs:

- **Non-atomic sibling writes**: a record persisted *before* its permission row, leaving a window where private content was retrievable workspace-wide. Invisible to tests; found by the write-path-integrity lens.
- **Gate-escapes**: type errors in generated code that passed both the typechecker (runs before generation) and the build (configured to ignore errors). Found by auditing the gates themselves.
- **Flagged-but-broken state**: records marked searchable whose index entry was deleted and never rebuilt: present in every count, absent from every search.
- **Promises with no code**: security-page claims with zero implementing lines.

## Install

**Claude Code**: copy the skill folder.

```bash
git clone https://github.com/apoorvjain25/production-audit.git
# personal (all projects)
cp -r production-audit/production-audit ~/.claude/skills/
# or per-project (shared with your team via the repo)
cp -r production-audit/production-audit your-project/.claude/skills/
```

**Everything else** (Cursor, Windsurf, Copilot, aider, raw API): paste [PROMPT.md](PROMPT.md). Same methodology, single file, zero install.

## Use

| Command | What you get |
|---------|--------------|
| `/production-audit` | full product audit, discovery only; nothing is modified |
| `/production-audit fix` | audit → fix in severity waves → verification loop until 2 clean passes |
| `/production-audit security` | one lens family at full depth |
| `/production-audit src/billing` | one subsystem through all 24 lenses |
| `/production-audit docs-vs-code` | every public claim verified against the implementation |

Works on any stack: web app, API, CLI, mobile, monorepo. The skill builds its inventory from *your* product's surfaces before it audits, so nothing is assumed about your architecture.

> **Heads up:** the full pipeline is thorough by design. Discovery on a real product produces hundreds of rows, and `fix` mode will happily run 10+ verification rounds. Scope it if you want a quick pass.

## Reading the report

| Tag | Means | Example |
|-----|-------|---------|
| `CRITICAL` | data loss, breach reachable today, broken core flow, crash on a primary path | cross-tenant cache leak |
| `HIGH` | claimed feature broken/missing, security weakness one precondition away, silent failure | payment-webhook failures swallowed |
| `MEDIUM` | degraded behavior, edge-case failure, real inconsistency | chart crashes on an empty dataset |
| `LOW` | minor bug, cosmetic defect, polish | missing favicon |
| `IMPROVEMENT` | a concrete, named upgrade, still no hedging | atomic decrement instead of check-then-act |

Rows are ordered CRITICAL → IMPROVEMENT and grouped by area (`SECURITY`, `AUTH`, `DATA`, `PERF`, `CONTENT`, ...) within each severity, so a team can fix top-down, row by row.

## What's in the box

```
production-audit/
├── SKILL.md                        # the skill: process, format, rules
└── references/
    ├── audit-angles.md             # 24 discovery lenses, each with a real example finding
    └── finding-taxonomy.md         # 18 defect classes + severity rubric + borderline calls
examples/
└── sample-report.md                # 30 anonymized rows + both convergence ledgers
PROMPT.md                           # the whole methodology in one paste-able file
```

## FAQ

<details>
<summary><b>How long does a full run take?</b></summary>

Hours, not minutes. That's the point. Discovery on a real product produces hundreds of rows across many passes, and `fix` mode routinely runs 10+ verification rounds. For a quick pass, scope it: `/production-audit src/billing` or `/production-audit security`.
</details>

<details>
<summary><b>Will it change my code?</b></summary>

Not unless you ask. The default run is discovery only: read everything, modify nothing. `fix` mode does edit, but every wave is gated on a green build + typecheck + lint + tests, and an over-reach review reverts any change that went beyond its finding.
</details>

<details>
<summary><b>What about false positives?</b></summary>

Every candidate finding is attacked before it's reported: is there a guard upstream? Is the check enforced elsewhere? Is that dead code actually unreachable? Findings that can't be pinned to a `file:line` or `URL + selector` are dropped. Plausible-but-wrong rows destroy trust in the whole list, so they don't make it.
</details>

<details>
<summary><b>Why is there no executive summary?</b></summary>

Summaries are where audits go to soften. "Overall the codebase is in good shape" tells you nothing actionable and quietly buries the rows that matter. Every row in this report stands alone (severity, location, defect, fix), so the list itself is the deliverable.
</details>

<details>
<summary><b>My product isn't a web app. Does this still work?</b></summary>

Yes. Phase 0 builds the inventory from whatever surfaces *your* product actually has: CLI commands, API endpoints, mobile screens, background jobs, docs. Lenses that don't apply (e.g. LLM quality with no LLM features) are skipped; everything else runs at full depth.
</details>

<details>
<summary><b>What's the difference between SKILL.md and PROMPT.md?</b></summary>

Same methodology, two packagings. <code>production-audit/SKILL.md</code> + its references install as a Claude Code skill, with the lens catalog and taxonomy loaded on demand. <code>PROMPT.md</code> is the whole thing flattened into one file you can paste into any other agent.
</details>

<details>
<summary><b>How do I know it actually converged instead of just stopping?</b></summary>

The skill keeps a pass ledger (pass number, lens used, new findings count) and is only allowed to stop when two consecutive passes from <i>different</i> lenses add zero new rows. Ten quiet sweeps of the same lens count as one angle, not ten. After fixes, the bar is two consecutive passes with zero CRITICAL/HIGH.
</details>

## Philosophy

> Trust what the code does, not what it's called.
> A short list means you didn't look hard enough.
> One quiet pass is not convergence.

## Origin

This skill was extracted from a real pre-launch audit of [Pulse](https://pulsehq.tech?utm_source=github&utm_medium=production-audit), a company brain built for the agent era. The 1,200-finding run this README opens with was our own codebase. We ran the loop until it converged, then open-sourced the methodology.

## Contributing

Found a defect class the taxonomy misses, or a lens that would have caught a bug in your product? PRs welcome: add the lens to [audit-angles.md](production-audit/references/audit-angles.md) with a one-line example finding, and keep [PROMPT.md](PROMPT.md) in sync.

## License

MIT. Use it, fork it, ship it.

---

*If this skill finds something scary in your codebase, that's the skill working. ⭐ the repo and tell someone what it caught.*
