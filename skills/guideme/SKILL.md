---
name: guideme
description: Plan and guide an integration or setup that Claude can't perform directly (provisioning cloud infra, wiring a device, standing up a tunnel, configuring a third-party service). Clarifies the request, researches the current path, writes a LOG + IMP plan to both the Obsidian vault and a local mirror, then guides execution one step at a time — Claude does the scripts/configs, the user does logins and console/device actions. Also resumes an existing plan when given a file or context.
argument-hint: <what you want enabled> | <path to an existing LOG/IMP to resume>
---

You are running `/guideme`. The user wants to accomplish something **you cannot do directly for them** — it lives outside this machine or behind a login/console/device they control (provisioning cloud infra, registering a domain, wiring hardware, configuring a SaaS, standing up a tunnel). Your job is to clarify it, research the real current path, capture it as durable memory (a LOG + an IMP), and then guide the user through execution step by step, doing every part you actually can.

Input: `$ARGUMENTS`

## First: is this a NEW request or a RESUME?

- **Resume** if `$ARGUMENTS` points at an existing plan — a vault note path, a local file path under `~/.claude/guideme/`, an integration name you can find there, or the user is clearly feeding you an existing LOG/IMP as context. → Read both the LOG and IMP, summarize where things stand (last completed step, next step, open questions), and jump to **Phase 4 (Guided execution)**.
- **New** otherwise (including empty `$ARGUMENTS` — then ask what they want to enable). → Start at **Phase 1**.

---

## Phase 1 — Clarify

The vaguer the request, the more you ask. The point is to converge on intent and the constraints that actually change the plan — not to interrogate.

Use the `AskUserQuestion` tool. Calibrate:
- **Vague** ("host my website on the internet") → several questions. What is the thing today (static files? a running server? a port?), expected traffic, budget ceiling, do they already have accounts (AWS? Cloudflare? a domain?), region, how long it needs to live, security/uptime expectations, preferred provider or "cheapest/simplest."
- **Specific** ("create a t3.micro in us-east-1 to serve this static site behind Cloudflare") → confirm the few genuine unknowns and move on.

Rules:
- Ask only what changes the plan. Don't ask things you can discover (inspect the local project/files yourself first).
- Surface unstated assumptions and let the user correct them, rather than asking open-ended questions.
- If several distinct approaches exist (e.g. EC2 vs. a static host vs. a tunnel from the local machine), note that now — you'll lay out the trade-off in Phase 2.
- Capture every answer; it becomes the top of the LOG.

## Phase 2 — Research, then choose a path

Do live web research before writing the plan — providers' consoles, CLIs, pricing, and gotchas drift, and the IMP must reflect *current* reality.

- Use `WebSearch` / `WebFetch` to verify the actual current steps, required CLI/console flows, pricing tiers, free-tier limits, common failure points, and security defaults.
- Prefer official docs; note the source and the date you checked for anything that could go stale (pricing, free-tier, API versions).
- If more than one viable path survives, present the trade-offs (cost, effort, reversibility, lock-in, how much *you* can automate vs. how much the user must click) and let the user pick. The chosen path and *why the others lost* both go in the LOG.

## Phase 3 — Write the LOG and the IMP

Write **both** documents to **both** locations, and keep the two copies byte-identical on every future update:

1. **Vault (canonical):** `GUIDES/<Integration Name>/LOG.md` and `GUIDES/<Integration Name>/IMP.md` via `mcp__obsidian__write_note`. (`<Integration Name>` Title Case, e.g. `EC2 Website Hosting`. If a folder of that name already exists with playbook docs, place `LOG.md`/`IMP.md` alongside them — don't disturb existing files.)
2. **Local mirror:** `~/.claude/guideme/<slug>/LOG.md` and `~/.claude/guideme/<slug>/IMP.md` via the `Write` tool (`<slug>` kebab-case of the same name). Create the folder if needed.

Both copies are standalone mirrors of the same content. **Do not put the vault path inside the local copy or vice versa** — each must read cleanly on its own (no cross-location links).

### LOG.md — the exploration record
Living, honest scratchpad. Append as you go; never rewrite history.

```markdown
# LOG — <Integration Name>

## Request
<the original ask, verbatim-ish>

## Clarifications (<date>)
<the Phase 1 Q&A, distilled — constraints, accounts already held, budget, region, etc.>

## Research findings (<date>)
<what you verified, with sources + date-checked for anything that can go stale>

## Options considered
- **<Chosen path>** — why it won
- <Other path> — why it lost
- ...

## Session notes
<dated running notes added during execution: what was tried, what worked, dead ends, surprises>
```

### IMP.md — the implementation plan
The concrete contract you execute against. One checkbox per step. Use `[ ]` for pending and `[x]` only for genuinely completed steps — **never mark a future/aspirational step done** (a ✅ or `[x]` means it actually happened).

```markdown
# IMP — <Integration Name>

**Goal:** <one sentence — definition of done>
**Chosen path:** <short name> (see LOG for why)

## Prerequisites
- <accounts, CLIs, credentials, local artifacts the user must have on hand>

## Steps

### [ ] 1. <step title>
- **Owner:** Claude | User
- **Do:** <exact commands, config, or console clicks>
- **Done when:** <acceptance criterion — how we know it worked>
- **If it fails:** <fork / fallback>

### [ ] 2. ...

## Open questions
- <anything unresolved that could change later steps>
```

**Owner rule:** mark a step **Claude** when it's something you can produce on this machine (write a script, generate config/IaC, craft a CLI command, edit local files). Mark it **User** when it requires a login, a web console, a credential entry, a hardware/device action, or anything off this machine. Default to doing the work yourself wherever possible; hand off only the parts that genuinely require the user.

After writing both docs, show the user the IMP and confirm before executing.

## Phase 4 — Guided execution

Walk the IMP **one step at a time**. For the current step:

1. State the step number, its owner, and exactly what's about to happen.
2. **If Owner = Claude:** do it now — write the script/config/file, prepare the exact command. Show the result. If a command must run against the user's account/device, hand them the precise command to run rather than assuming it ran.
3. **If Owner = User:** give precise, ordered instructions (which page, which button, what to paste). Wait.
4. Then the user either:
   - says **"next"** / confirms success → mark the step `[x]` in the IMP, add a one-line note to the LOG `## Session notes`, and move to the next step.
   - **reports an issue / asks a question / gives an update** → help debug, adjust. When the real working path differs from what the IMP said, **rewrite that step (and any downstream steps it affects) to match what actually worked**, and record what happened in the LOG. The docs must always describe the path that *actually works*, not the original guess.

On every step transition, **sync both copies** (vault via `patch_note`/`write_note`, local via `Edit`/`Write`) so they stay identical. If the integration involves a git repo, also commit per completed step (one commit per step) — but many `/guideme` steps are console/device actions with nothing to commit, so only commit when there's an actual code/config change in a repo.

Keep going until every step is `[x]` and the Goal's "done when" is satisfied. Then write a short closing note in the LOG (final working state, anything to remember for teardown or renewal — e.g. "instance costs ~$X/mo, stop it when done", domain/cert renewal dates).

## Rules

- You are the planner and the hands for everything on this machine; the user is the hands for everything off it. Never pretend you performed an off-machine action.
- One step at a time in Phase 4 — don't dump the whole sequence and walk away.
- The LOG and IMP are the memory. Anything learned mid-execution that changes the path goes into them immediately, in both locations.
- Keep both mirror copies identical and each self-contained (no link from one to the other's location).
- Prefer reversible, cheap, simple paths unless the user asked otherwise; call out cost and lock-in.
- Don't invent placeholder credentials, account IDs, or values to make a command "look complete" — leave a clearly marked blank for the user to fill, or have them supply it.
