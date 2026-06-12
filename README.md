# 🔍 production-audit

**The Claude Code skill that audits your product until it stops finding things — then proves it.**

Most "audit my code" prompts do one pass, find 15 issues, and write you a reassuring summary. This skill found **1,200+ real findings** on a production B2B SaaS — then took **14 verification passes** after the fixes before it could honestly say "done."

```
/production-audit
```

## What makes it different

**1. Two convergence loops, not one pass.**
A single audit pass only finds what that pass's framing makes visible. This skill keeps auditing from *different angles* — attack-class, claim-vs-code, data-shape extremes, lifecycle, failure modes, gate-escapes — and only stops discovery when **two consecutive diverse passes find nothing new**. After fixes, a second loop runs until **two consecutive passes find zero CRITICAL/HIGH**. "We checked" becomes "we converged."

**2. Every finding is attacked before it's reported.**
Each candidate finding gets a refutation attempt against the real code: is there a guard upstream? Is it enforced elsewhere? Is that dead code actually unreachable? Plausible-but-wrong findings destroy trust in an audit — they don't make this list.

**3. The flat list IS the report.**
No executive summary. No "overall, the codebase is in good shape." Every row is pinned and actionable:

```
[CRITICAL] [SECURITY] src/api/documents/route.ts:84 — share-link content served after revocation; revokedAt never checked — add revokedAt IS NULL to the access query
[HIGH] [DATA] src/import/processor.ts:142 — document row committed before its permission row, non-atomically; content is live and unprotected in the gap — wrap both writes in one transaction, permission first
[HIGH] [CONTENT] landing/security.html — page claims "AES-256 encryption at rest" — no encryption configured anywhere in the storage layer; implement it or remove the claim
[MEDIUM] [RELIABILITY] src/jobs/webhook-sender.ts:51 — failed deliveries dropped with no retry or dead-letter path — add backoff retry + DLQ
```

Rules the skill enforces on itself: every row has `file:line` or `URL + selector` (no location ⇒ dropped), no "consider/might/could", no padding, and an honest `TRUNCATED AT …` line if it runs out of context instead of a fake wrap-up.

**4. Claims are code.**
Every specific promise on your landing page, README, docs, and security page gets verified against the implementation. "We never train on your data", "undo within 5 minutes", "AES-256" — each is either backed by code or it's a HIGH finding.

## The kinds of bugs it catches that tests don't

From real runs:

- **Non-atomic sibling writes** — a record persisted *before* its permission row, leaving a window where private content was retrievable workspace-wide. Invisible to tests; found by the write-path-integrity lens.
- **Gate-escapes** — type errors in generated code that passed both the typechecker (runs before generation) and the build (configured to ignore errors). Found by auditing the gates themselves.
- **Flagged-but-broken state** — records marked searchable whose index entry was deleted and never rebuilt: present in every count, absent from every search.
- **Promises with no code** — security-page claims with zero implementing lines.

## Install

**Claude Code** — copy the skill folder:

```bash
git clone https://github.com/YOUR_USERNAME/production-audit-skill.git
# personal (all projects)
cp -r production-audit-skill/production-audit ~/.claude/skills/
# or per-project (shared with your team via the repo)
cp -r production-audit-skill/production-audit your-project/.claude/skills/
```

**Everything else** (Cursor, Windsurf, Copilot, aider, raw API) — paste [PROMPT.md](PROMPT.md). Same methodology, single file, zero install.

## Use

```
/production-audit                  # full product audit, discovery only
/production-audit fix              # audit → fix in severity waves → verification loop
/production-audit security        # scoped: one lens, full depth
/production-audit src/billing     # scoped: one subsystem, all lenses
/production-audit docs-vs-code    # scoped: every public claim vs implementation
```

Works on any stack — web app, API, CLI, mobile, monorepo. The skill builds a feature inventory from *your* product's surfaces (routes, docs, marketing pages, jobs, integrations) before it audits, so nothing is assumed about your architecture.

> **Heads up:** the full pipeline is thorough by design. Discovery on a real product produces hundreds of rows, and `fix` mode will happily run 10+ verification rounds. Scope it if you want a quick pass.

## What's in the box

```
production-audit/
├── SKILL.md                        # the skill — process, format, rules
└── references/
    ├── audit-angles.md             # 12+ discovery lenses (loaded on demand)
    └── finding-taxonomy.md         # defect classes + severity rubric
PROMPT.md                           # single-file version for any agent
```

## Philosophy

> Trust what the code does, not what it's called.
> A short list means you didn't look hard enough.
> One quiet pass is not convergence.

## License

MIT — use it, fork it, ship it.

---

*If this skill finds something scary in your codebase, that's the skill working. ⭐ the repo and tell someone what it caught.*
