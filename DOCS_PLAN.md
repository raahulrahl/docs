# Docs restructure plan

**Companion to:** [`DOCS_AUDIT.md`](./DOCS_AUDIT.md)
**Branch where this plan was drafted:** `docs/audit-fixes`
**Last updated:** 2026-05-17

The audit found 4 mechanical bugs (now fixed and pushed) and 4 structural problems. This document is the phased plan for the structural ones.

---

## The anchor

Every phase optimizes for one goal:

> *A new dev lands on docs.getbindu.com, follows the "Get Started" arc, has a working agent on their machine in under 10 minutes, and knows where to read next.*

If a proposed change doesn't serve this, it's polish — defer it. If it does, it's load-bearing.

---

## Phase map at a glance

| Phase | What | Risk | Time | Ships as |
|---|---|---|---|---|
| **0** | Mechanical bugs | — | done | merged |
| **1** | New "Get Started" arc — kill the consolidated guide, promote orphans, write real Config Reference | **High** — public URLs change | ~4–6h | 1 PR |
| **2** | Split `protocol.mdx` into a Protocol Reference group | **Med** — anchor links change | ~3–4h | 1 PR |
| **3** | Voice pass on `learn/` pages that read like reference dumps | Low | ~2–3h | 1 PR |
| **4** | Examples consistency pass — frontmatter, numbering, import paths verified | Low | ~2h | 1 PR |
| **5** | Housekeeping — delete dead assets, decide `sapthame/`, refresh changelog cadence | Low | ~1h | 1 PR |

Total: ~12–16 hours across 5 PRs. Each PR is independently shippable; if we stop after Phase 1+2 the docs are already dramatically better.

---

## Open decisions before Phase 1 starts

Two structural calls affect every page rewritten downstream. Both should be locked before we touch Phase 1.

### Decision 1: `agent_config.json` vs `config = {dict}`

**The reality on disk:** out of 22 Python examples, exactly **one** uses `agent_config.json`. The rest define `config = {...}` as a Python dict in the same file as the handler. `examples/beginner/agno_simple_example.py:37` even carries an explicit comment:
> *"Infrastructure configs (storage, scheduler, sentry, API keys) are now automatically loaded from environment variables."*

**The reality in docs:** ~8 pages describe `agent_config.json` as the canonical pattern — `storage/overview`, `scheduler/overview`, `skills/introduction/overview`, consolidated guide (in 3 places), `join-the-internet-agents`, `create-bindu-agent/overview`.

**The fork:**
- **(A) Match the code.** Dict + env vars is canonical. JSON is removed or framed as "if you prefer a JSON file, here's how to load it." This is what the codebase already implies.
- **(B) Resurrect JSON.** Add `agent_config.json` support back into every example, treat dict-form as legacy.

**Default I'd take if you don't pick:** (A). The code has voted with its feet; making docs match is honest. The `with open("agent_config.json") as f:` pattern is one optional helper in the Config Reference, not the front door.

### Decision 2: Tab structure

Current "Developer Journey" tab has 7 groups: Get Started, Architecture, Gateway, gRPC sidecar, Identity & Trust, Action & Memory, Internet of Agents, Production Reliability.

**Problem:** "Action & Memory" + "Internet of Agents" + "Production Reliability" are each 3-page groups, but the boundary between them is fuzzy. A reader looking for "how do I retry on failure" has to know it's under "Production Reliability" not "Action & Memory."

**The fork:**
- **(A) Flatten** — collapse three groups into one "Capabilities" group, ~9 pages, alphabetical.
- **(B) Keep groups, fix names** — rename to one-word categories: Identity / Storage / Network / Reliability / Payment.
- **(C) Leave as-is** for now, revisit if user feedback says it's confusing.

**Default I'd take:** (C). The current grouping isn't broken enough to block on. Phase 3's voice pass solves more of the real "I can't find X" pain than re-shuffling tabs would.

---

## Phase 1 — New Get Started arc

**Goal:** A first-time reader has a working agent in 5 minutes, then knows the next 3 pages to read.

### New nav (proposed)

```
Get Started
├── what-is-bindu              (keep — already fine, code fix landed)
├── install                    (promote orphan how-to/install.mdx)
├── your-first-agent           (NEW — 10-line agno walkthrough, copy-runs)
├── config-reference           (NEW — the actual config dict)
└── next-steps                 (NEW — "now read X / Y / Z" — 1 screen)
```

### File operations

| Operation | Source | Target | Why |
|---|---|---|---|
| **Promote** | `bindu/how-to/install.mdx` (orphan, 192 lines) | `bindu/get-started/install.mdx` | Currently invisible in nav |
| **Promote** | `bindu/introduction/key-concepts.mdx` (orphan, 315 lines) | merge content into `what-is-bindu.mdx` or split into 2 conceptual pages | "Key Concepts" is what users search for |
| **Promote** | `bindu/create-bindu-agent/deploy.mdx` (orphan, 523 lines) | `bindu/get-started/deploy.mdx` | Largest orphan, real deploy content |
| **Create** | — | `bindu/get-started/your-first-agent.mdx` | New — verified copy-runs from `examples/beginner/agno_simple_example.py` |
| **Create** | — | `bindu/get-started/config-reference.mdx` | New — documents the actual config dict + env vars |
| **Delete** | `bindu/introduction/bindu-consolidated-guide.mdx` (898 lines) | — | All 5 sections move to dedicated pages |
| **Delete** | `bindu/introduction/join-the-internet-agents.mdx` (orphan, 325 lines) | — | Duplicates consolidated guide |
| **Delete** | `bindu/introduction/getting-help.mdx` (orphan, 137 lines) | — | Replace with a "Getting Help" callout in `next-steps.mdx` |
| **Rename** | `bindu/create-bindu-agent/configuration.mdx` (installer prompts) | `bindu/create-bindu-agent/installer-reference.mdx` | Stop calling installer-prompt docs "Configuration" |
| **Delete** | `bindu/create-bindu-agent/overview.mdx` (orphan, 98 lines) | — | Already merged into consolidated guide content |

### What `your-first-agent.mdx` should look like (sketch)

Take `examples/beginner/agno_simple_example.py` verbatim. Walk through it in 4 beats:

1. **The whole thing (15 lines).** Show the working file. Reader can copy and run.
2. **What just happened.** 3 sentences pointing at agent / config / handler / `bindufy()`.
3. **Make it yours.** 3 micro-tasks: change the prompt, swap the model, add a tool.
4. **What's next.** One link to Config Reference, one to Examples, one to Gateway.

This is the page where "agent live in 2 minutes" stops being a marketing claim and becomes a verified instruction.

### What `config-reference.mdx` should look like (sketch)

Two sections, no fluff:

1. **The required keys** — `author`, `name`, `description`, `deployment.url`, `deployment.expose`. Each gets one paragraph + a one-line example.
2. **The optional keys + env-var equivalents** — table format:

```
| Key                          | Env var               | Default     | What it does           |
|------------------------------|-----------------------|-------------|------------------------|
| storage.url                  | STORAGE__URL          | memory://   | Storage backend        |
| scheduler.url                | SCHEDULER__URL        | memory://   | Task scheduler backend |
| capabilities.push_notifications | (none)             | false       | Enable webhooks        |
| skills                       | (none)                | []          | Skill folder paths     |
...
```

Source of truth: grep `app_settings` and `bindufy()` arg parsing in `bindu/penguin/`. This page should be **autogeneratable** in the long run — for now, hand-written but cross-checked against `bindu/penguin/config_validator.py`.

### Public URL impact

Pages that move get a new URL. Three pages have significant inbound risk:
- `/bindu/introduction/bindu-consolidated-guide` — only ~3 months old, but if anything in the wild links here it'll 404. Add a Mintlify redirect.
- `/bindu/create-bindu-agent/configuration` — renamed. Redirect.
- `/bindu/how-to/install` — promoted, not deleted. Redirect.

Mintlify supports redirects in `docs.json` under `redirects`. We add 3 entries.

### Done criteria

- [ ] `mintlify broken-links` passes
- [ ] Every code block in `your-first-agent.mdx` and `config-reference.mdx` was actually run (not just copied)
- [ ] No orphan files in `bindu/introduction/`, `bindu/how-to/`, `bindu/create-bindu-agent/` (verify with the nav-vs-disk diff script in the audit appendix)
- [ ] Redirects added for the 3 moved URLs

---

## Phase 2 — Protocol Reference split

**Goal:** Replace the 4,691-line `protocol.mdx` with a navigable reference, and remove the CLI banner.

### New nav

Add a new tab or a new group under "Docs" tab:

```
Protocol Reference
├── overview              (~200 lines — what A2A / AP2 / X402 are, why they exist)
├── tasks-and-states      (~600 lines — TaskState, lifecycle, TaskStatus, events)
├── messages-and-artifacts (~500 lines — Parts, Message, Artifact, communication flow)
├── agent-card-and-skills (~600 lines — AgentCard, Skill, capabilities, discovery)
├── negotiation           (~400 lines — Bindu-specific negotiation extensions)
├── security-and-trust    (~500 lines — TrustLevel, security schemes, DID-flavored sections)
├── payment-mandates      (~600 lines — AP2 payment models, mandates, credit system)
└── notifications         (~400 lines — push notifications, webhook config)
```

8 pages, each in the 200–700 line range. None are too long to navigate, each has a clear scope.

### File operations

| Operation | What |
|---|---|
| **Split** | `bindu/concepts/protocol.mdx` (4691 lines) → 8 files above |
| **Delete** | The "Getting Started" section at lines 4608–4692 (the CLI ASCII banner). It does not belong in a type reference. |
| **Move** | The `## Overview` section becomes the new `overview.mdx` for the group |
| **Keep** | A redirect from `/bindu/concepts/protocol` to the new `/protocol/overview` |

### Anchor link impact

`protocol.mdx` has 71 H3s and 15 H2s. Every external link with `#anchor` will break unless the anchor lands on the right new page. Mitigation: each new page keeps the H2/H3 ids it inherits; we add page-level redirects only.

### What's the "Concepts" group then?

Currently "Concepts" contains `protocol.mdx` (the dump) + `task-first-and-architecture.mdx` (a merged file). After Phase 2:

- Concepts shrinks to one page: `task-first-and-architecture.mdx`, optionally split back into two pages or kept merged.
- Protocol Reference is its own group/tab.
- Concepts becomes a place for *narrative* pages only — "how to think about this" — not type references.

### Done criteria

- [ ] `protocol.mdx` is deleted
- [ ] All 8 new pages render in nav
- [ ] No section is over 700 lines
- [ ] No CLI banner anywhere in the doc body
- [ ] Anchor IDs preserved where possible
- [ ] Redirects added

---

## Phase 3 — Voice pass on `learn/`

**Goal:** Bring the `learn/` pages that read like reference up to the bar set by DID, Auth, and Gateway.

### Pages that need a voice intro

| Page | Problem | What's missing |
|---|---|---|
| `learn/storage/overview.mdx` | Opens with "Persist tasks, context, and artifacts…" — no story | 3-sentence narrative opener |
| `learn/scheduler/overview.mdx` | Opens with "Queue and distribute tasks…" — no story | 3-sentence narrative opener |
| `learn/notification/overview.mdx` | Opens with "The Long-Running Task Notification System gives clients a way…" — institutional voice | Reframe as: "your task takes 10 minutes. Webhook this back when it's done." |
| `learn/negotiation/overview.mdx` | "Capability-based agent selection for intelligent orchestration" — reads like a slide title | Concrete scenario: planner picking between 3 agents |
| `learn/observability/overview.mdx` | OK voice, just thin on the "why" | Add the "production is silent until it screams" framing |
| `learn/health-metrics/overview.mdx` | Reasonable but minimal | Verify it's not just `/health` returning 200 |

### Pages that already work — model for the rest

- `learn/did/overview.mdx` — "milk-truck problem"
- `learn/authentication/overview.mdx` — "answering a stranger"
- `learn/payment/introduction.mdx` — opens with the agents-paying-agents framing
- `gateway/overview.mdx` — Tokyo population example

Use these as templates. Each one starts with a real-world scenario, names the problem in concrete terms, then shows the solution.

### Voice-pass template (3 paragraphs to add at the top of each page)

```
Para 1: A concrete scenario the reader has actually hit. Don't say "your task may fail." Say "you call an external API. It returns 503. What happens next?"

Para 2: Name the trap. What goes wrong if you handle this poorly? Be specific. ("Without retry, every transient blip becomes a failed task. Users see errors that wouldn't have existed if you'd just waited 200ms and tried again.")

Para 3: Name the solution in one line. ("Bindu retries failed task steps with exponential backoff. You configure the policy; Bindu handles the rest.")

Then the existing content takes over.
```

### Done criteria

- [ ] Every `learn/*/overview.mdx` opens with 3 paragraphs in the scenario → trap → solution pattern
- [ ] No page opens with "X is a system that…" or "X gives clients a way to…"
- [ ] Read each page out loud — if it sounds like a slide deck, rewrite

---

## Phase 4 — Examples consistency

**Goal:** Every example follows the same template; every code block is verified runnable.

### Frontmatter standardization

Every example file should have:
```mdx
---
title: "1.4 Research Assistant"
description: "A web-search agent built with Agno and DuckDuckGo, exposed via Bindu."
tags: ["beginner", "agno", "search"]
---
```

Currently only `title` is consistent. `description` is missing on ~80% of examples. `tags` are missing universally.

### Numbering audit

Spot-checked in the audit: `examples/beginner/ag2-agent.mdx` has title `"1.7 Question Answering Agent"` — file name says ag2, title says Question Answering. Pick one and align both.

Reproduce the table for all 30+ examples. Either the title matches the file or both get renamed.

### Code-block verification

For each example:
- [ ] Import path is `from bindu.penguin.bindufy import bindufy` (already verified in Phase 0)
- [ ] `bindufy(config, handler)` signature (already verified in Phase 0)
- [ ] Config dict has the required keys: `author`, `name`, `description`, `deployment`
- [ ] If the example uses an env var, it's documented in the "Environment Setup" section
- [ ] The "Run" command actually starts the agent locally (spot-check 5 of them by running them)

### Section template (proposed)

Standardize on:
1. **Title + one-line summary** (frontmatter)
2. **One-paragraph intro** (what this agent does, who it's for)
3. **Code** (the full file)
4. **How it works** (3 short bullets — instructions / tools / model)
5. **Dependencies + .env**
6. **Run** (the command)
7. **Try it** (3 example inputs)
8. **Example API calls** (optional, only for examples that warrant it)
9. **Frontend setup** (optional)

Some examples have all 9 sections, some have 4. Pick a canonical structure and enforce.

### Done criteria

- [ ] Every example has `description` and `tags` in frontmatter
- [ ] Every example's numbered title matches its filename theme
- [ ] At least 5 examples spot-checked by actually running them
- [ ] Section ordering consistent

---

## Phase 5 — Housekeeping

Fast pass to remove cruft.

| Action | What |
|---|---|
| Delete | `openapi_v0.yaml` (61 KB, untracked, unused since Feb 2026) |
| Delete | `assets/bidu-idea.gif` (never referenced) |
| Delete | `assets/key-concepts.gif` (never referenced) |
| Delete | `assets/sunflower_2.jpeg` (never referenced — `1.jpeg` and `3.jpeg` are used) |
| Decide | `sapthame/introduction/overview.mdx` — it's a great story (multi-agent challenge), but the whole `sapthame/` directory has its own top-level dir and no tab uses it. Either (a) move into `bindu/concepts/` and add to nav, or (b) delete. |
| Decide | Changelog — last entry `v2026.12.5` is dated 2026-03-19. Either resume updates (~1 entry per major release) or remove the tab. A stale tab is worse than no tab. |
| Update | `README.md` in docs repo root — replace generic Mintlify template with a real "how to contribute to these docs" |
| Add | `.gitignore` entry for `openapi_v0.yaml` if we want to keep historical copies locally but not commit |

### Done criteria

- [ ] No untracked files in `docs/` (besides `node_modules` if it exists)
- [ ] Every file in `assets/` is referenced somewhere
- [ ] Changelog decision made and acted on
- [ ] `sapthame/` decision made and acted on

---

## Risk register

Things that could go wrong, ordered by likelihood × impact:

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| Phase 1 URL changes break inbound links from blog posts, social, AI search citations | High | Med | Add redirects in `docs.json`; keep old slugs in a `redirects` table |
| Phase 2 anchor changes break deep links to specific protocol types | Med | Low | Keep H2/H3 ids stable; redirect at page level |
| Voice rewrite drifts from "Bindu voice" to "Claude voice" | Med | Med | Each rewritten page reviewed against `learn/did/` and `gateway/overview` as voice anchors |
| Splitting `protocol.mdx` reveals undocumented types or contradictions | Med | Low | Cross-check each section against `bindu/common/types.py` at split time |
| Decision 1 (JSON vs dict) goes the wrong way and we rewrite again | Low | High | Get explicit Raahul ✓ before Phase 1 starts |

---

## Sequencing rationale

Why phase order is **1 → 2 → 3 → 4 → 5**:

- **Phase 1 first** because the broken Get Started arc affects every new visitor. Highest user-pain-per-line-of-content.
- **Phase 2 second** because the Protocol page is the deepest content debt; addressing it after Phase 1 means a clean Get Started funnels readers into a clean Reference.
- **Phase 3 third** because the voice problems are real but only matter once the bones are right.
- **Phase 4 fourth** because examples are mostly fine; this is a polish pass, not a rescue.
- **Phase 5 last** because cruft removal is satisfying but doesn't block anything.

If we can only ship 2 phases this quarter: **1 and 2**.
If we can only ship 1 phase this quarter: **1**.

---

## What I'd need from you to start Phase 1

1. **A call on Decision 1** (JSON vs dict). Default I'll take: dict + env vars.
2. **A call on Decision 2** (tab structure). Default I'll take: leave as-is.
3. **A green light to delete** `bindu-consolidated-guide.mdx`, `join-the-internet-agents.mdx`, `getting-help.mdx`, `create-bindu-agent/overview.mdx` (the orphans + the dumping ground).
4. **30 seconds to confirm** the new nav sketch under "Phase 1 — New nav" is what you want.

If those are all yes/defaults, Phase 1 is ~4–6 hours of mechanical work I can do start-to-finish in a single PR.

---

*Plan written without modifying any code. Implementation begins only on Phase-by-Phase approval.*
