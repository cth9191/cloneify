# cloneify

**Turn "I want to clone X" into a single paste-ready prompt that builds it.**

`cloneify` is a [Claude Code](https://claude.com/claude-code) skill that conducts a
four-phase, human-gated pipeline for cloning a piece of software. You stay in the
driver's seat — every phase stops for your go-ahead — and the pipeline composes two
other skills rather than reinventing them.

The real deliverable is **Phase 3's goal-prompt**: one self-contained block you paste
into a fresh Claude Code session to actually build the clone. Phases 0–2 exist to make
that prompt *correct* — scoped, researched, and adversarially hardened.

---

## The four phases

| Phase | Name | What happens |
|-------|------|--------------|
| **0** | **Scope** | One-question-at-a-time interview: platform, must-have features, hard constraints (the *reason* to clone), stack, explicit out-of-scope. → `scope.md` |
| **1** | **Research** | Invokes a **deep-research** skill to understand the target deeply — real feature set, technical approach, existing clones, and the landmines most likely to sink the build. → `research.md` |
| **2** | **Plan** | Invokes **grill-me-codex**: Claude interviews you to lock every open decision, then **OpenAI Codex adversarially reviews the plan** in a read-only sandbox until `APPROVED`. → `PLAN.md` + `PLAN-REVIEW-LOG.md` |
| **3** | **Handoff** | Emits `GOAL-PROMPT.md` — a self-contained build prompt (full plan inlined) plus a matching `/goal` completion condition so Claude Code builds autonomously until it's met. |

**Gates are hard stops.** Each phase produces an artifact, shows it to you, asks the
gate question, and waits. You can always say "go back" and revise the prior phase.

---

## Why it works

- **Scope before research** — research has a target, and the plan has guardrails.
- **Research before planning** — decisions rest on sourced facts, not vibes.
- **Grill + cross-model review before building** — the #1 failure is building the
  wrong thing (fixed by the grill); the #2 is a plan that *sounds* right but breaks
  (fixed by a *different model*, Codex, attacking it).
- **A stand-alone goal-prompt** — the build session doesn't need this run's files;
  the plan and scope are inlined into the prompt.

---

## Prerequisites

cloneify is a **conductor** — it orchestrates other tools. You need:

- **Claude Code** with skills enabled.
- A **`deep-research` skill** available in your environment (Phase 1 invokes it by
  name). Any deep-research harness that returns a sourced report works; swap the
  Phase 1 call for your own if you use a different one.
- The **[grill-me-codex](https://github.com/) skill** (Phase 2 invokes it by name).
- The **[Codex CLI](https://github.com/openai/codex)** installed and authenticated
  (`codex login`) — grill-me-codex uses it for the adversarial plan review.
- For the optional autonomous build: **Claude Code v2.1.139+** with `/goal`.

> Without a `deep-research` skill and `grill-me-codex` (+ Codex CLI) present,
> Phases 1–2 won't run. cloneify does not bundle them.

---

## Install

Drop `SKILL.md` into your Claude Code skills directory:

```
~/.claude/skills/cloneify/SKILL.md
```

Then invoke it:

```
/cloneify
```

…or just describe wanting to clone something ("I want a local version of X",
"let's clone Superhuman") and Claude will trigger it.

---

## Run folder

All artifacts for a run live beside where you'll build, in your workspace — **not**
inside the skill folder:

```
<run-root>/                 # e.g.  <cwd>/<slug>-local/  — the target repo / build dir
└── clone-run/              # process artifacts, beside the app code
    ├── STATE.md            # resume across sessions
    ├── scope.md            # Phase 0
    ├── research.md         # Phase 1
    ├── PLAN.md             # Phase 2
    ├── PLAN-REVIEW-LOG.md  # Phase 2 — the full Codex argument
    └── GOAL-PROMPT.md      # Phase 3 — the paste-ready build prompt
```

`STATE.md` makes runs resumable — re-invoking cloneify picks up from the last
completed phase.

---

## Example

> **You:** `/cloneify` → "WisprFlow, but 100% local on Windows"
>
> cloneify interviews you (platform, features, the ≤1.5s latency constraint, your
> GPU), deep-researches how WisprFlow works and the local-ASR landmines, grills you
> on the open decisions (which cleanup model, which ASR runtime, injection method),
> has Codex tear the plan apart until it's sound, and hands you one prompt that
> builds the whole thing — GPU-stack-first so the risky part fails fast, not last.

---

## Credits & license

- cloneify is released under the **MIT License** (see [LICENSE](LICENSE)).
- Phase 2 builds on **[grill-me-codex](https://github.com/)**, which in turn builds
  on **Matt Pocock's `grill-me`** (MIT). The grilling methodology lineage is his;
  the cross-model Codex review and the four-phase clone pipeline are additions.
