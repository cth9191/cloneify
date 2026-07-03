---
name: cloneify
description: >-
  End-to-end conductor for cloning a piece of software. Walks the user through
  four gated phases — scope → deep research → plan (grilled + Codex-reviewed) →
  a paste-ready build prompt — composing the deep-research and grill-me-codex
  skills along the way. Use this whenever the user wants to clone, rebuild,
  reimplement, or make their own version of an existing app or tool ("clone
  WhisperFlow", "build my own Superhuman", "I want a local version of X",
  "reverse-engineer this app into a spec", "let's clone something"), or asks for
  a repeatable research→plan→build pipeline for replicating software. Trigger it
  even when the user just describes wanting to recreate an app's functionality
  without saying the word "clone". Invoke via /cloneify.
---

# cloneify

A guided pipeline that turns "I want to clone X" into a single paste-ready prompt
that builds it. You are the conductor: you run four phases, **stop at every gate
for the user's go-ahead**, and compose two existing skills (`deep-research`,
`grill-me-codex`) rather than reinventing them.

The real deliverable is **Phase 3's goal-prompt** — one self-contained block the
user pastes into a fresh Claude Code session to actually build the clone. Phases
0–2 exist to make that prompt *correct*: scoped, researched, and hardened.

## Why gates matter

Each phase is expensive and consequential (research burns tokens; grilling takes
real user time; the plan steers everything downstream). The user must stay in the
driver's seat. **Never auto-advance.** Finish a phase, show what you produced,
then ask the explicit gate question and wait. If the user wants to revise the
previous phase instead of moving on, loop back — don't force forward.

## Run folder & state

Pick a short kebab-case slug for the target (e.g. `whisperflow`, `superhuman`,
`raycast`).

**Where the run folder lives — resolve this BEFORE writing any file.** Artifacts
must land in the **user's project, next to where they'll actually build**, NOT
inside this skill's own base directory. The base directory named in the skill
invocation is where the skill *lives*, never where its output goes. Concretely:

- The run folder is `<run-root>/clone-run/`, where `<run-root>` is a directory in
  the user's workspace that will become (or sit beside) the **target repo they run
  `/goal` in**.
- Default `<run-root>` to `<cwd>/<slug>-local/` (a dedicated subfolder of the
  current working directory). At the **start of Phase 0**, state where you intend
  to put it and let the user redirect — e.g. "I'll put the run + build here:
  `<cwd>/<slug>-local/`. Good, or somewhere else?" Honor their answer.
- **Never** write to a path under `~/.claude/skills/` or wherever this SKILL.md
  resides. If you can't determine the cwd, ask — don't fall back to the skill dir.

Layout (all paths below are relative to the resolved `<run-root>`):

```
<run-root>/                 # e.g.  <cwd>/whisperflow-local/  — target repo / build dir
└── clone-run/              # process artifacts, beside the app code
    ├── STATE.md            # which phase is done; lets you resume across sessions
    ├── scope.md            # Phase 0 output
    ├── research.md         # Phase 1 output (from deep-research)
    ├── PLAN.md             # Phase 2 output (from grill-me-codex)
    ├── PLAN-REVIEW-LOG.md  # Phase 2 Codex argument (from grill-me-codex)
    └── GOAL-PROMPT.md      # Phase 3 output — the paste-ready prompt
```

At the **start of every invocation**, resolve `<run-root>` (above), then check
whether `<run-root>/clone-run/STATE.md` exists. If it does, read it and resume from
the last completed phase instead of starting over. If the user names a new target,
start fresh. Update `STATE.md` after each phase completes (record phase, date,
one-line status).

---

## Phase 0 · SCOPE

Goal: nail down *what* we're cloning and *how far*, so research has a target and
the plan has guardrails.

1. Ask first: **"What are we cloning?"** Get the target app/tool by name (or a
   description if it's niche/internal).
2. Then interview for scope — **one focused question at a time**, recommending an
   answer for each so the user can just confirm. Cover:
   - **Platform / runtime** — Windows / macOS / Linux / web / cross-platform?
   - **Must-have features** — the 3–6 that define the clone. What's the core loop?
   - **Hard constraints** — the non-negotiables that are the *reason* to clone
     rather than just use the original (e.g. "100% local / offline", "no
     subscription", "open-source deps only", "runs on my GPU").
   - **Stack preferences** — language/framework, or "you choose."
   - **Explicit out-of-scope** — what we are deliberately *not* building in v1.
     This is as important as the must-haves; it keeps the plan honest.
3. Write `scope.md` with sections: Target, Platform, Must-have features,
   Hard constraints, Stack, Out of scope. Keep it tight and skimmable.

**Gate:** show `scope.md`, then ask: *"Scope locked. Ready for the deep research
phase? (It'll dig into how the original works, its real feature set, and the
gotchas — costs some time/tokens.)"* Wait for go-ahead.

---

## Phase 1 · RESEARCH

Goal: understand the target deeply enough to clone it — not vibes, sourced facts.

1. Invoke the **deep-research** skill via the Skill tool. Feed it a question built
   from `scope.md`, e.g.:
   > "Research <target> for a clone project. Cover: what it actually does and its
   > core UX loop; the full user-facing feature set; how it works technically
   > (architecture, key dependencies, models/APIs, platform tricks); known
   > limitations and the hard problems a clone must solve; any existing
   > open-source clones and what they got right/wrong. Constraints we care about:
   > <hard constraints from scope>."
2. When deep-research returns, distill its report into `research.md` focused on
   what the *builder* needs: feature inventory, the technical approach, the
   landmines (the 3–5 things most likely to sink the build), and any decisions
   the research surfaced that the user will need to make in planning.
3. Summarize back to the user conversationally — lead with "here's what I found",
   highlight the landmines, and flag the open decisions planning will resolve.

**Gate:** ask: *"That's the research. Want to start the planning phase? I'll grill
you on the open decisions, then have Codex adversarially review the plan before we
lock it."* Wait for go-ahead.

---

## Phase 2 · PLAN

Goal: turn scope + research into a hardened, build-ready spec.

1. Invoke the **grill-me-codex** skill via the Skill tool. Hand it the context it
   needs: point it at `<run-root>/clone-run/scope.md` and `research.md`, and tell it
   the plan should target the must-have features under the hard constraints, with
   the out-of-scope list respected. Let grill-me-codex run its two acts — the
   one-question-at-a-time interview, then the Codex review loop to APPROVED.
2. Ensure the locked plan lands as `<run-root>/clone-run/PLAN.md` (and the review
   argument as `PLAN-REVIEW-LOG.md`). If grill-me-codex wrote `PLAN.md` elsewhere
   (e.g. repo root), copy it into the run folder so the run is self-contained.

**Gate:** ask: *"Plan's locked and Codex-approved. Move to the execution phase? I'll
hand you the single prompt to paste into a fresh Claude Code session to build it."*
Wait for go-ahead.

---

## Phase 3 · HANDOFF

Goal: emit the paste-ready goal-prompt **plus a matching `/goal` completion
condition** — so the user can paste, set the finish line, and let Claude Code
build autonomously until it's met.

Write `GOAL-PROMPT.md`. It carries everything the build session needs without
access to this run, so **inline the essentials** rather than referencing files
the new session won't have open. Use this structure:

```markdown
# Build goal: clone of <target>

> HOW TO RUN THIS (two moves in a fresh Claude Code session, in your target repo):
> 1. Paste everything below this line as your first message.
> 2. Then set the finish line so Claude keeps working until it's done:
>    /goal "<completion condition — see the bottom of this file>"

---

You are building a <one-line description> — <hard constraints, e.g. "fully local,
Windows-only">. Build it from the locked spec below. Work in de-risking order,
verify as you go, and show each milestone working before moving on.

## What we're building (feature scope)
- <must-have feature 1>
...

## Hard constraints (non-negotiable)
- <constraint 1>
...

## Explicitly out of scope (v1)
- <out-of-scope item 1>
...

## The plan
<paste the full, Codex-approved PLAN.md contents here>

## How to proceed
1. Confirm the environment / dependencies first (a doctor-style check).
2. Build in the plan's de-risk order; spike the riskiest unknowns first.
3. After each milestone, run it and show it working before moving on.

---

## Recommended /goal condition

Claude Code's `/goal` keeps working turn-after-turn until a small evaluator model
judges your condition met. The evaluator reads **what the build session surfaces
in the conversation** — it does NOT run commands itself — so phrase the condition
around something Claude's own output demonstrates. Derive it from the plan's
acceptance criteria. Example:

    /goal "the app builds and runs, and a self-test / acceptance run printed in
    this conversation shows <the plan's verifiable pass signal, e.g. 'ALL CHECKS
    PASS'> with every must-have feature exercised"

Keep it to one measurable end state + how Claude should prove it. `/goal` needs
Claude Code v2.1.139+, an accepted workspace-trust dialog, and hooks enabled; it
auto-resumes across `--continue`/`--resume`, so a long build can span sessions.
```

Fill the `<...>` placeholders from `scope.md` and `PLAN.md`. For the `/goal`
condition, pull the plan's actual acceptance check (e.g. a self-test or doctor
command whose output names a clear pass string) so the evaluator has something
concrete to read.

Then present it to the user: show the prompt, say where it's saved
(`<run-root>/clone-run/GOAL-PROMPT.md`), and walk the two-move handoff — paste the
prompt into a fresh Claude Code session **opened in `<run-root>` (the target repo)**,
then run the `/goal` line so it builds until the condition holds. The app code
builds in `<run-root>`; the reference artifacts sit beside it in `clone-run/`.

Mark `STATE.md` complete.

---

## Operating principles

- **One question at a time** in interviews; recommend an answer so the user can
  confirm fast. Don't dump a questionnaire.
- **Gates are hard stops.** Produce → show → ask → wait. The user can always say
  "go back" and you revise the prior phase.
- **Compose, don't reinvent.** Research is `deep-research`'s job; grilling and the
  Codex review are `grill-me-codex`'s job. You sequence them and own the artifacts.
- **Everything to disk.** Each phase leaves a file in the run folder so the run is
  resumable and the process is visible (useful if this is being recorded/taught).
- **The goal-prompt must stand alone.** Inline the plan and scope into it; the
  build session won't have this run's files.
