# Docs audit — `docs.getbindu.com`

**Date:** 2026-05-17
**Scope:** `/Users/raahuldutta/Documents/GetBindu/docs/` (Mintlify)
**Read:** every section's headline file + voice spot-checks + cross-checks against `bindu/` runtime code.
**Verdict:** Your instinct is right. The docs as written compile clean (Mintlify says no broken links), but the *structure* is fighting you — there's a dumping-ground page papering over orphaned files, the canonical "Get Started" path has factually wrong code, and the same content exists in 2–3 places with minor edits. New users land in a maze.

---

## TL;DR — the four real problems

1. **The "Get Started" path is broken on day one.** The landing code sample (`bindufy(agent, config, handler)`) doesn't match the real Python signature (`bindufy(config, handler)`). A reader who copy-pastes will fail immediately. Same wrong signature appears in the consolidated guide and the orphaned `join-the-internet-agents.mdx`.
2. **A 898-line "consolidated guide" is hiding a navigation problem.** Someone took 3–4 orphan files and concatenated them into `bindu-consolidated-guide.mdx` instead of fixing the sidebar. The originals (`join-the-internet-agents.mdx`, `key-concepts.mdx`, `create-bindu-agent/overview.mdx`, `how-to/install.mdx`) are still on disk — they just don't appear in `docs.json`. They drift independently.
3. **The "Protocol" page is a 4,691-line reference dump with a CLI ASCII banner at the bottom.** It's the type-system source of truth pasted into a doc. No reader gets through it. The Bindu-specific endpoints listed at the end (`/a2a`, `/agent/info`) **don't exist** in the actual code.
4. **The "Configuration" doc isn't a configuration doc.** It's a list of installer (cookiecutter) prompts. The real runtime `config` dict — the thing every example writes — has no reference page. One step in that file even shows mismatched title vs. body.

Fix these four and you have a coherent doc set. The voice in the well-written pages (DID, Authentication, Gateway, gRPC overview) is genuinely good — you don't need to rewrite, you need to restructure.

---

## 1. Orphans on disk (real pages, not in nav)

These files exist in `docs/` but are NOT referenced by `docs.json`. They're invisible from the sidebar, mostly reachable only by a direct URL.

| File | Lines | Status |
|---|---|---|
| `bindu/concepts/architecture.mdx` | 341 | Concatenated into `task-first-and-architecture.mdx` — original left behind |
| `bindu/concepts/task-first-pattern.mdx` | 233 | Same — merged into the "and" file, original left behind |
| `bindu/create-bindu-agent/overview.mdx` | 98 | Body re-pasted into `bindu-consolidated-guide.mdx` |
| `bindu/create-bindu-agent/deploy.mdx` | **523** | Largest orphan. Not in nav anywhere. |
| `bindu/how-to/install.mdx` | 192 | Re-pasted into consolidated guide |
| `bindu/introduction/getting-help.mdx` | 137 | Just orphaned |
| `bindu/introduction/join-the-internet-agents.mdx` | 325 | Re-pasted into consolidated guide; original still here |
| `bindu/introduction/key-concepts.mdx` | 315 | Re-pasted into consolidated guide; "Key Concepts" is exactly what a new user searches for |
| `sapthame/introduction/overview.mdx` | — | Lives under its own top-level dir, no tab uses it |

**Asset orphans** (in `assets/` but never referenced):
- `bidu-idea.gif`, `key-concepts.gif`, `sunflower_2.jpeg`

**OpenAPI orphan:** `openapi_v0.yaml` (61 KB, Feb 2026) — unused, only `openapi.yaml` is wired in.

**Implication:** Mintlify's `broken-links` check passes because nothing links *to* these pages. Google can still index them. A reader who finds `key-concepts.mdx` via search lands on a page with no sidebar context and stale content. This is the worst kind of orphan — silently divergent.

---

## 2. The "consolidated guide" anti-pattern

`bindu/introduction/bindu-consolidated-guide.mdx` is **898 lines** and contains, in order:

1. "Join the Internet of Agents" — also in `join-the-internet-agents.mdx`
2. "Built for the Internet of Agents" — overlaps `key-concepts.mdx`
3. "Task Lifecycle and States" — overlaps `concepts/task-first-and-architecture.mdx`
4. "Create Bindu Agent Overview" — also in `create-bindu-agent/overview.mdx`
5. "Install & Setup" — also in `how-to/install.mdx`

Each section uses a "Situation / Task / Action / Result" template ("Let me share a quick story about why this matters…"). The repetition makes the page feel padded; readers stop scrolling around section 3.

It also breaks linking discipline: the "Learn More" accordion at the bottom links to `/bindu/create-bindu-agent/overview`, `/bindu/introduction/key-concepts`, `/bindu/learn/authentication` — three of those targets are orphans. So the canonical doc page sends readers to pages that don't appear in the sidebar.

**Why this happened (my guess):** someone reorganized the nav, but rather than delete the obsolete pages, they pasted their bodies into one file and left the originals as a safety net. The safety net is now the problem.

---

## 3. Factually wrong code & endpoints

### 3a. `bindufy` signature is wrong on the landing page

Real signature (`bindu/penguin/bindufy.py:628`):
```python
def bindufy(config, handler, run_server=True, key_dir=None, launch=False)
```

Docs say:
| File | Line | Claim |
|---|---|---|
| `bindu/introduction/what-is-bindu.mdx` | 146 | `bindufy(agent, config, handler)` ❌ |
| `bindu/introduction/bindu-consolidated-guide.mdx` | 166 | `bindufy(my_agent, config, agent_handler)` ❌ |
| `bindu/introduction/join-the-internet-agents.mdx` | (orphan) | Same wrong pattern ❌ |

Real examples in `examples/beginner/agno_simple_example.py:76` use `bindufy(config, handler)` — i.e. the docs disagree with the code Raahul actually ships. A first-time user copies the landing snippet → `TypeError`.

### 3b. Fake HTTP endpoints

`protocol.mdx:4664–4692` advertises:
```
A2A:            http://localhost:3773/a2a       ← does not exist
Agent Info:     http://localhost:3773/agent/info ← does not exist
```

Real routes (`bindu/server/applications.py:185–289`):
- A2A is `POST /` (the bare root), not `/a2a`
- Agent metadata is `/.well-known/agent.json`, not `/agent/info`
- Real routes also include `/agent/skills`, `/did/resolve`, `/health`, `/metrics`, `/agent/negotiation`, and the x402 payment endpoints

The "Visit `http://localhost:3773/agent/info`" instruction at the end of `protocol.mdx` will 404.

### 3c. Import-path inconsistency across examples

Out of 22 Python examples, three different import statements are in use:

| Pattern | Where | Works? |
|---|---|---|
| `from bindu.penguin.bindufy import bindufy` | ~18 examples (majority) | ✅ |
| `from bindu.penguin import bindufy` | `1.7 Question Answering`, `2.5 Cybersecurity Newsletter`, `3.3 AG2 Research Team` | ✅ (re-export) |
| `from bindu.bindufy import bindufy` | `1.1 Echo Agent` | ❌ **broken** — no `bindu/bindufy.py` exists |

The first example a user clicks (Beginner → Echo Agent) won't run. Pick one canonical import and apply globally.

### 3d. `agent_config.json` is over-claimed

The docs describe `agent_config.json` as the canonical config in 8+ places (storage, scheduler, paywall pages, plus consolidated guide). The repo has exactly **one** `agent_config.json` left (in `examples/multilingual-collab-agent/`). Every other example builds the config as a Python dict literal in the main file. Live `agno_simple_example.py` even has a comment: *"Infrastructure configs (storage, scheduler, sentry, API keys) are now automatically loaded from environment variables."*

So the docs describe a config-as-JSON workflow the codebase has already moved away from. Either resurrect the JSON pattern or rewrite the docs to match the dict + env-var reality.

---

## 4. File-level bugs

### 4a. Broken frontmatter — `bindu/learn/retry/overview.mdx`
```mdx
---

## title: "Retry Mechanism" description: "Automatic retry logic..."

---
```
Missing line break and the closing `---`. Mintlify will render `## title: "Retry Mechanism" description: ...` as a literal H2 in the page body, and the page title in the browser tab will fall back to the filename. **This is shipped to production right now.**

### 4b. Mismatched step title/body — `bindu/create-bindu-agent/configuration.mdx:118–132`
```mdx
<Step title="Identity & Cryptography" icon="key">
## Observability & Monitoring
```
Step is labeled "Identity & Cryptography" but its body talks about Phoenix/Jaeger/Langfuse. Either the step was renumbered without updating the title, or the body was swapped in.

### 4c. ASCII banner ending — `bindu/concepts/protocol.mdx:4630–4684`
The "Getting Started" section at the bottom of the protocol doc reproduces the CLI splash screen — ~50 lines of `}{}{}{}{` borders, an ASCII sunflower, then version info, "⭐️⭐️⭐️ Support Open Source ⭐️⭐️⭐️", Discord link. That's the **last thing** a reader sees after a 4,691-line type reference. It looks like a debug paste, not a doc.

### 4d. Single-file mega-doc — `bindu/concepts/protocol.mdx` itself
4,691 lines, 15 H2s, 71 H3s. Sections covered: TaskState, Negotiation, TrustLevel, Parts, Communication Types, Task Management, Context Management, Negotiation again, Security, Notifications, Payment Models, Mandates, Credit System, Trust & Identity, Agent Discovery, plus the broken Getting Started. This is the entire `types.py` rendered as prose. It belongs as either (a) an autogenerated reference page sourced from the types, or (b) split into ~6 pages under a "Protocol Reference" group. A reader can't navigate it.

---

## 5. What the structure looks like vs. what would work

Current "Developer Journey" tab as declared in `docs.json:92–164`:

```
Get Started
  what-is-bindu            ← landing + wrong code sample
  bindu-consolidated-guide ← 898-line dumping ground
  configuration            ← actually installer prompts, not config reference

Architecture
  task-first-and-architecture ← merged file, originals orphaned
  protocol                    ← 4,691-line type dump

Gateway          ✅ well organized, good voice
gRPC sidecar     ✅ well organized, good voice
Identity & Trust ✅ DID page is great; Auth page is great
Action & Memory  (Skills + Storage + Scheduler)
Internet of Agents (Tunneling, Negotiation, Payment, Notification)
Production Reliability (Retry, Observability, Health-metrics)
```

**The asymmetry is jarring.** Gateway and gRPC are well-structured multi-page sections (overview → quickstart → reference). DID and Auth are excellent narrative pages. But the first three things a new user sees ("Get Started" → "Architecture") are the worst-organized part of the whole site. The reader who survives them never gets to the good content.

What would actually serve the "she just built an agent, she wants it live in 2 minutes" persona from `what-is-bindu.mdx`:

```
Get Started
  what is bindu       (the why — keep, fix code sample)
  install             (the how — uv add bindu — promote the orphan)
  your first agent    (10-line agno_simple_example.py walkthrough)
  config reference    (the actual config dict, not installer prompts)

Concepts
  task-first model    (~250 lines from current "and" file)
  protocol overview   (~400 lines — what A2A/AP2/X402 mean, no types)
  endpoints           (real /.well-known/agent.json etc — what the server actually exposes)

Protocol Reference   (split protocol.mdx into 5–7 pages)
  task & states
  messages & artifacts
  agent card & skills
  negotiation
  security & trust
  payment mandates
  notifications

Gateway (no changes)
gRPC sidecar (no changes)
Capabilities (rename "Identity & Trust" + "Action & Memory" + "Internet of Agents" — too many tabs for related topics)
Production
```

Two structural moves do most of the work: **(a)** promote the orphaned `install` / `key-concepts` / `deploy` into a real "Get Started" arc, delete the consolidated guide; **(b)** split `protocol.mdx` into a sub-section with its own group in nav. Everything else is cleanup.

---

## 6. Voice consistency — mixed, but the good pages are *really* good

The doc voice from your memory (`feedback_doc_voice.md`: second-person, problem-first, concrete numbers) is followed beautifully in:

- `bindu/learn/did/overview.mdx` — the milk-truck story
- `bindu/learn/authentication/overview.mdx` — "deciding whether to answer a stranger"
- `bindu/gateway/overview.mdx` — "Tokyo population poem" worked example
- `bindu/grpc/overview.mdx` — "the infrastructure trap"
- `bindu/learn/payment/introduction.mdx` (skimmed — looked consistent)

These pages don't need rework. They're the model for the rest.

The voice **drifts** on:

- `bindu/introduction/bindu-consolidated-guide.mdx` — Situation/Task/Action/Result template repeated 5+ times reads like a corporate template
- `bindu/concepts/protocol.mdx` — pure reference voice, OK for reference but the file is mis-categorized as a concept
- `bindu/learn/notification/overview.mdx` — opens with "Long-Running Task Notification System gives clients a way…" — institutional voice, no story
- `bindu/learn/negotiation/overview.mdx` — same problem; opens cold with "Capability-based agent selection…"
- `bindu/learn/storage/overview.mdx`, `scheduler/overview.mdx` — terse, no narrative hook

It's not that these are bad — they're functional reference. But sitting next to the DID/Auth/Gateway pages, they feel like a different writer. A reader notices.

---

## 7. Stale / suspicious content

- **Changelog last entry: `v2026.12.5` dated 2026-03-19.** Today is 2026-05-17 — eight weeks of silence. Either no releases happened (unlikely, see git tags) or the changelog wasn't updated. Worth a one-line check.
- **`changelog/` filenames use `v` prefix** (`v2026.12.5.mdx`). Your release-process memory says tags are `yyyy.week.day` *without* the `v`. The git tags themselves use `v` — so the memory was stale, but it's worth re-confirming what's canonical, because the URL slug `changelog/v2026.12.5` is a permanent public link.
- **`index.mdx` is 729 lines of inline Tailwind/React** — it's a custom landing, not a Mintlify-default. That's fine, but it duplicates almost all messaging from `what-is-bindu.mdx`. When you update one, you have to update both.
- **`README.md`** in the docs repo root is generic Mintlify template ("To preview the documentation changes locally…"). Either replace with a real "how to contribute to these docs" or delete it from the audit's mental model (it's not user-facing).

---

## 8. What's actually working — keep this

So you don't think the whole thing needs a rewrite:

- **`docs.json` itself is well-organized as a config.** Tabs are sensible, SEO/OG tags are thorough, fonts/colors consistent.
- **Mintlify is healthy.** `mintlify broken-links` is clean. No syntax errors blocking the build.
- **Gateway, gRPC, DID, Auth, Payment** sections are publishable as-is.
- **Examples have great breadth** (33 examples across beginner/specialized/advanced). The problem is uniformity, not scope.
- **Roadmap voice is excellent** ("Think about how you pay for stuff online today. You click a button…"). Keep that writer.

---

## 9. Recommended order of operations (when we're ready to edit)

Roughly cheapest-to-highest-value first. Nothing here yet — just the order:

1. **Fix the broken landing code sample** — `what-is-bindu.mdx:146` and `consolidated-guide.mdx:166` (`bindufy(config, handler)`). 5 minutes. Stops first-impression breakage.
2. **Fix the broken frontmatter in `retry/overview.mdx`.** 2 minutes.
3. **Fix the wrong endpoints in `protocol.mdx`** (`/a2a` → `/`, `/agent/info` → `/.well-known/agent.json`). 5 minutes.
4. **Standardize the import path in all Python examples** — pick `from bindu.penguin import bindufy` or `from bindu.penguin.bindufy import bindufy`. Fix the broken `from bindu.bindufy import bindufy` in `echo-agent.mdx`. 15 minutes.
5. **Delete `bindu-consolidated-guide.mdx`.** Promote `key-concepts.mdx`, `how-to/install.mdx`, `create-bindu-agent/overview.mdx`, `create-bindu-agent/deploy.mdx`, `join-the-internet-agents.mdx` into the "Get Started" group in `docs.json`. Resolve content drift one section at a time (canonical wins).
6. **Delete `bindu/concepts/architecture.mdx` and `task-first-pattern.mdx`** — they're orphan originals of the merged file. Or, better, undo the merge and put each back into nav as its own page.
7. **Split `protocol.mdx` into 5–7 reference pages** under a new nav group. Replace it with a short conceptual page + autogen-from-types reference.
8. **Write a real Configuration Reference** — the dict shape every example uses, the env-var fallbacks. Move installer-prompt doc to `installer-reference.mdx` or delete (it overlaps the README of `create-bindu-agent`).
9. **Voice pass on `learn/notification`, `learn/negotiation`, `learn/storage`, `learn/scheduler`** — give each a 3–5 sentence story-opener that matches the DID/Auth pages.
10. **Delete `openapi_v0.yaml`, unused assets, and `sapthame/`** (or restore `sapthame/` to nav — it's a good story and currently invisible).

Items 1–4 are 30 minutes of mechanical fixes that stop the bleeding. Items 5–7 are the actual restructure. Items 8–10 are polish.

---

## Appendix — verification commands used

```bash
# nav vs disk diff
python3 -c "..." # in chat — produced orphan list above

# broken links
cd docs && mintlify broken-links   # PASSES — but doesn't catch orphans

# import paths across examples
for f in examples/**/*.mdx; do
  grep -m1 "from bindu" "$f"
done

# real bindufy signature
grep -A 6 "^def bindufy" Bindu/bindu/penguin/bindufy.py

# real HTTP routes
grep "_add_route" Bindu/bindu/server/applications.py
```

---

*Audit complete. No code or doc files were modified.*
