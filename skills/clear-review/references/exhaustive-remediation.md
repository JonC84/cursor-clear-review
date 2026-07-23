# CLEaR Exhaustive Remediation Workflow

How to turn a large CLEaR report into a durable plan that a cheaper model can execute safely, one wave per fresh chat, without losing context or "fixing" load-bearing behavior.

**Use this only when the Plan of Attack is too big for one session** — an exhaustive whole-codebase pass, many findings, or cross-file invariants. For a small scoped diff, skip all of this and execute the report's waves directly.

The governing principle: **the plan is the memory, not the chat.** Every wave runs in a clean context seeded only with the written plan.

## Artifacts

1. **Master triage plan** — one file in the project's docs dir (e.g. `docs/clear-remediation-triage.md`). It triages every finding into `Fix | Preserve-invariant | Verify-intent | Won't-fix`, records the resolution of every Verify-intent question, and lists the tracks.
2. **Per-track plan** — one file per independent track (e.g. `plans/clear-<track>-plan.md`), containing an ordered `- [ ] Wave N` checklist. Tracks are typically: security/access, data-integrity, backend-cost, refactors.

## Step 1 — Resolve intent before any code

List every `Verify-intent` finding as a plain question. Get the user's decision on each, record it in the master plan, and reclassify (→ Fix / Preserve-invariant / Won't-fix). **No track execution begins while an intent question is open** — this is the main guard against defect-elimination bias.

## Step 2 — Decompose into tracks

Split the plan into independent tracks. Each track gets its own `brainstorming` design → `writing-plans` plan. Never write one mega-spec. Order tracks by value-first / risk-last (quick wins → data integrity → cost → refactors), unless live exposure forces security first.

**Give each finding one primary track.** If a finding has both a safe slice and a behavior-changing slice (e.g. delete dead code *and* de-duplicate live logic), split it into two rows — a safe row in the earlier track, a risky row in the later track — instead of tagging it `X/Y`. Dual tags force every design chat to stop and ask which slice belongs where; single tags let it run.

**Default rule for any dual-tagged finding you still encounter** (so a design chat never has to ask): `X/Y` means only the safe / dead-delete slice belongs to the earlier / lower-risk track; behavior-changing slices defer to the higher-risk track. When designing track X, include only the slices safe at X's tier and defer the rest to their other track with a one-line note. Default to this stricter split; ask only on genuine ambiguity.

## Step 3 — Standard wave steps (baked into every execution wave)

1. **Re-verify** every `file:line` before editing — the review is a snapshot; line numbers drift.
2. **Honor the Class:** `Fix` → remediate · `Preserve-invariant` → change the mechanism, keep the stated Invariant · `Won't-fix` → skip.
3. **Minimal diff** + CLEaR principles (clean/no-comments, lightweight, efficient, robust). No drive-by edits.
4. **Safety net:** add/confirm the characterization test or written smoke checklist the plan specifies, passing on current code first (`test-driven-development`).
5. **Verify:** run the plan's stated test + lint/build commands; show output as evidence (`verification-before-completion`).
6. **Commit** — one wave = one commit/PR, revertible on its own.
7. **Self-review handoff:** launch a strong-model `requesting-code-review` on the wave diff, address any blockers, then **STOP**. Do not start the next wave.

## Step 4 — Chat prompts (copy-paste, one variable set per track)

Substitute `{MASTER}` (master plan path), `{TRACK}` (track name), and `{PLAN}` (that track's plan path). Have Step-2 design record the project's concrete test/lint/build commands in the plan header so the execution prompt stays command-agnostic and blank-free.

### Design chat — run once per track

```
Read {MASTER} in full. Design the {TRACK} track only: run /brainstorming then /writing-plans to produce {PLAN}. One wave per {TRACK}-tagged finding, ordered low-risk-first, each written as a self-contained brief following the Standard wave steps (incl. the step-7 self-review handoff). For a finding tagged with multiple tracks, include only the slice safe at this track's tier and defer the rest to its other track with a one-line note — do not stop to ask. Record the project's concrete test + lint/build commands in the plan header. Render the waves as a "- [ ] Wave N: <ID> — <summary>" checklist. Preserve every stated Invariant. List the waves and STOP for my review — write no code.
```

### Execution chat — run once per wave (repeat in a fresh chat until the checklist is done)

```
Read {PLAN} and {MASTER}. Execute ONLY the next unchecked wave, following the Standard wave steps exactly: re-verify line numbers, honor the finding Class/Invariant, minimal diff + CLEaR, add the specified safety net, run the plan's stated test + lint/build commands and show evidence, commit, tick the wave's checkbox in the plan, then launch a strong-model requesting-code-review on the diff. Then STOP. Do not start another wave.
```

## Tiering

Run Fix/Preserve-invariant waves on a capable-but-cheap model; the step-7 diff review by a strong model is what makes that safe (reviewing a diff is far cheaper than regenerating it). Any `High`-risk wave → strongest tier + mandatory review, isolated PR, never batched.
