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
6. **Commit** — one wave = one commit/PR, revertible on its own. Tick **this step's** checkbox in the plan when the commit lands. **Do not tick the parent Wave N checkbox yet.** **Do not otherwise edit the plan** — only tick checkboxes for the wave you are executing; never rewrite, reorder, or revert plan prose or tags (they are intentional, not stray content from a prior wave).
7. **Review at the wave's Review tier** (assigned at design time — see Tiering): **R0** = automated gate only (test + lint/build green **and** a diff-scope check that only the declared `file:line`s changed; no LLM); **R1** = cheap-model `requesting-code-review`; **R2** = strong-model `requesting-code-review`. **Escalate to R2** if any check fails, the diff strays outside its declared scope, or a lower tier flags anything. Address blockers. When review clears, tick **this step's** checkbox, then — **and only then** — tick the parent **Wave N** checkbox (all Standard steps 1–7 must be `[x]` first). **STOP**. Do not start the next wave.
8. **Emit the next handoff.** Print the prompt to run the next wave using the "Handoff format" below (always labeled for a **fresh chat**). If every wave is now ticked, emit the **Track-completion handoff** (below) instead.

**Checkbox discipline:** Tick each Standard wave step (`1.`…`7.`) **as soon as that step completes** (progressive — visible along the way). The parent `- [ ] Wave N` checkbox stays unchecked until **every** step for that wave is `[x]`. Never mark a wave done with open steps.

## Step 4 — Chat prompts (copy-paste, one variable set per track)

Substitute `{MASTER}` (master plan path), `{TRACK}` (track name), and `{PLAN}` (that track's plan path). Have Step-2 design record the project's concrete test/lint/build commands in the plan header so the execution prompt stays command-agnostic and blank-free.

### Design chat — run once per track

```
Read {MASTER} in full. Design the {TRACK} track only: run /brainstorming then /writing-plans to produce {PLAN}. One wave per {TRACK}-tagged finding, ordered low-risk-first, each written as a self-contained brief following the Standard wave steps (incl. the step-7 tiered review). Assign each wave a Review tier from its Class × Risk × safety-net (R0 = Low-risk pure Fix — dead-delete / hygiene / dep bump — with an authoritative safety net; R1 = Low-risk Fix touching live code with a passing characterization test; R2 = any Preserve-invariant, Med/High risk, security/access/data/cost surface, or a wave with no real safety net). For a finding tagged with multiple tracks, include only the slice safe at this track's tier and defer the rest to its other track with a one-line note — do not stop to ask. Record the project's concrete test + lint/build commands in the plan header. Each wave brief must include a one-line **Manual smoke:** field (1–3 ≤2-minute Desktop/Mobile checks for Pause mode). Render the waves as a "- [ ] Wave N (Rx): <ID> — <summary>" checklist, and render each wave's Standard wave steps as a "- [ ]" sub-checklist so execution can tick completed steps. Preserve every stated Invariant. List the waves and STOP for my review — write no code. After the list, print the **spec/plan written handoff** (3-way menu below): Manual (NEW chat required) + Automated · Continuous + Automated · Pause (NEW chat optional for both Automated options — paste here or in a NEW chat). Include the exact Execution prompt and both Orchestrator kickoff prompts filled for this track.
```

### Spec/plan written handoff (print after the wave list)

```
Approved? Choose how to execute:

1) Manual — open a NEW chat and paste:
   <Execution chat prompt>
   (fresh chat required: the parent session is the executor)

2) Automated · Continuous — paste here *or* in a NEW chat:
   <Orchestrator Continuous kickoff>
   (subagents provide isolation; new chat optional)

3) Automated · Pause — paste here *or* in a NEW chat:
   <Orchestrator Pause kickoff>
   (same; pause after each cleared wave for testing/analysis)
```

### Execution chat — run once per wave (Manual mode; repeat in a fresh chat until the checklist is done)

```
Read {PLAN} and {MASTER}. Execute ONLY the next unchecked wave, following the Standard wave steps exactly: re-verify line numbers, honor the finding Class/Invariant, minimal diff + CLEaR, add the specified safety net, run the plan's stated test + lint/build commands and show evidence, commit. Tick each Standard wave step checkbox in the plan as soon as that step completes (progressive). Do not tick the parent Wave N checkbox until steps 1–7 are all [x] — tick Wave N only after Review (step 7) clears. Do not otherwise edit the plan — never rewrite/reorder/revert its prose or tags. Then review the diff at that wave's Review tier (R0 = automated gate only, no LLM; R1 = cheap-model requesting-code-review; R2 = strong-model requesting-code-review), escalating to R2 on any failed check, out-of-scope diff, or flagged issue. Then STOP and do not start another wave. Finally, print the next handoff (Standard wave step 8): "Next wave (N of M) — open a NEW chat and paste:" followed by this exact same prompt; if every wave is now ticked, emit the Track-completion handoff instead (the next track's Design prompt for a NEW chat, or — if this was the last track — all-tracks-complete plus a recommended fresh exhaustive CLEaR re-review).
```

### Handoff format (Manual vs Automated)

**Manual:** each wave ends by handing you the next prompt for a **new chat** — never continue in place. State the reason inline:

> ▶ Next wave (N of M) — open a **NEW** chat and paste the prompt below. The fresh context is deliberate: the plan file is the memory, and long sessions degrade cheaper models and blur wave isolation.

A Manual handoff never means "keep going here." The step-7 STOP is hard; step 8 only prints text for you to carry to the next clean chat.

**Automated:** the parent is only an orchestrator; **subagents** provide isolation. Opening a new chat is **optional** — paste the Continuous or Pause kickoff in the design chat *or* a NEW chat. Prefer staying put when the wave list was just approved; open a NEW chat only if the design session is huge/noisy.

### Track-completion handoff (continuity between tracks)

A finished track is a handoff too, not a dead end. When a track's last wave is ticked and its reviews clear, emit:

- `✅ <TRACK> track complete`.
- The **next track** in the master plan's order, then `▶ Start it in a NEW chat:` + that track's **Design chat prompt**.
- If it was the **last** track: `🎉 All tracks complete — <PROJECT> remediation finished`, then recommend a fresh **exhaustive CLEaR re-review** to confirm the cleanup held, plus any deferred strongest-model waves as isolated PRs.

This keeps at the track boundary the same fresh-chat discipline step 8 keeps at the wave boundary.

## Automated mode (optional: subagent orchestration)

Offer Automated at the **spec/plan written handoff** (alongside Manual), not only in a plan appendix. Subagents give wave isolation automatically — each runs in a clean context — so an orchestrator can drive the loop. **Opening a new chat is optional** for Automated (paste in the design chat or a NEW chat); Manual still requires a NEW chat because the parent *is* the executor.

Pick a cadence:

- **Continuous** — after each wave's Review tier clears, immediately start the next unchecked wave through track completion. Still hard-STOP on any failed check / review flag / out-of-scope diff. Still pause for approval before starting the *next track*.
- **Pause** — after each wave's Review tier clears, print a short summary (files, commit SHA, Review tier result), then **Quick manual checks** (1–3 concrete ≤2-minute Desktop/Mobile steps confirming the wave's Invariant / happy path still works — prefer the wave brief's `Manual smoke:` line when present; otherwise derive from Invariant + changed files). Then STOP with `▶ Wave N done — reply continue / skip / abort (if broken, fix here first while context is fresh)`. Do not start the next wave until the user replies. Use this for testing/analysis between waves.

Failures always hard-STOP in both modes. Pause is *between cleared waves*, not instead of Review tiers. Quick checks are happy-path smoke, not a full regression suite (`npm test` already ran).

At design time, each wave brief should include a one-line **`Manual smoke:`** field (1–3 steps) so Pause does not invent checks cold.

**Sequencing is mandatory.** Waves commit to the same branch and each re-verifies line numbers against the just-changed tree, so run them one at a time: dispatch → await → review → (Continuous: next | Pause: wait). Never run two waves of one track concurrently on the same branch.

Orchestrator loop (parent session):

1. Dispatch an **execution subagent** for the next unchecked wave, on that wave's Tier model (cheap for Fix/QW); its prompt is the Execution chat prompt (includes progressive step ticks; Wave N stays unchecked until steps 1–7 are done).
2. **Await** it. On any test/lint/review failure, STOP and surface — do not proceed.
3. **Review at the wave's Review tier:** R0 → automated checks only, no review subagent; R1 → dispatch a cheap-model review subagent; R2 → dispatch a strong-model review subagent (`requesting-code-review`). Escalate to R2 on any failure or out-of-scope diff. When review clears, ensure **step 7** is `[x]`, then ensure the parent **Wave N** is `[x]` (only if steps 1–7 are all `[x]`).
4. **Checkbox gate (mandatory before Continuous next or Pause “done”):** open `{PLAN}` and confirm Wave N's Standard steps **1–7 are all `[x]`** and the parent Wave N is `[x]`. If the wave is `[x]` but any step is still `[ ]`, fix the ticks (tick missing steps or untick the wave) and STOP. If any step is still open, do not proceed.
5. **Continuous:** if gate passes, immediately start the next unchecked wave. **Pause:** if gate passes, print Quick manual checks, then STOP for `continue` / `skip` / `abort` (user may fix in-session if a check fails).
6. Repeat until every wave is ticked (or the user aborts), then report a summary. **On track completion, offer to proceed to the next track but pause for approval** — a new track needs its own design + sign-off before any code; never auto-run it.

**Parallelism** is only safe across genuinely independent waves or independent tracks, each in its **own git worktree/branch** merged sequentially (`using-git-worktrees` / `best-of-n-runner`). Parallel commits to one branch will collide — never parallelize waves within a track on one branch.

### Orchestrator kickoff — Continuous

```
Orchestrate the {TRACK} track from {PLAN} (+ {MASTER}) via subagents, mode=Continuous. For each unchecked wave sequentially: (1) dispatch an execution subagent on the wave's Tier model with the Execution chat prompt — it must tick each Standard wave step as that step completes and must NOT tick parent Wave N until steps 1–7 are all [x]; (2) await it — on any test/lint/review failure STOP and report; (3) review at the wave's Review tier — R0: automated checks only, no review subagent; R1: dispatch a cheap-model code-review subagent; R2: dispatch a strong-model code-review subagent — escalating to R2 on any failure or out-of-scope diff; when review clears, ensure step 7 then Wave N are [x]; (4) Checkbox gate: confirm in {PLAN} that Wave N steps 1–7 and Wave N are all [x] — if not, fix ticks or STOP; (5) if gate passes, immediately start the next unchecked wave — do not wait for me between waves. Never run waves in parallel on this branch. When all waves are ticked, emit the Track-completion handoff and pause for approval before any next-track design (do not start it automatically).
```

### Orchestrator kickoff — Pause

```
Orchestrate the {TRACK} track from {PLAN} (+ {MASTER}) via subagents, mode=Pause. For each unchecked wave sequentially: (1) dispatch an execution subagent on the wave's Tier model with the Execution chat prompt — it must tick each Standard wave step as that step completes and must NOT tick parent Wave N until steps 1–7 are all [x]; (2) await it — on any test/lint/review failure STOP and report; (3) review at the wave's Review tier — R0: automated checks only, no review subagent; R1: dispatch a cheap-model code-review subagent; R2: dispatch a strong-model code-review subagent — escalating to R2 on any failure or out-of-scope diff; when review clears, ensure step 7 then Wave N are [x]; (4) Checkbox gate: confirm in {PLAN} that Wave N steps 1–7 and Wave N are all [x] — if not, fix ticks or STOP; (5) after gate passes: print a short wave summary (files, commit SHA, Review tier result), then print **Quick manual checks** — 1–3 concrete ≤2-minute Desktop/Mobile steps a human can run to confirm the wave's Invariant still holds and nothing obvious broke (use the wave brief's Manual smoke: line when present; otherwise derive from Invariant + changed files; prefer happy-path over rare failure repro). Then STOP with: "▶ Wave N done — reply continue / skip / abort (if broken, fix here first while context is fresh)". Do not start the next wave until I reply continue (or skip to leave it unchecked and move on). Never run waves in parallel on this branch. When all waves are ticked (or I abort), emit the Track-completion handoff / status and pause for approval before any next-track design.
```

## Tiering

Two independent tiers per wave: the **execution tier** (which model writes the diff) and the **Review tier** (how expensive the diff review is). Keep both proportional to the wave's risk — **a review should not cost more than the work it guards.**

Run Fix/Preserve-invariant waves on a capable-but-cheap model. The review is what makes that safe, but not every wave needs the strong model:

- **R0 — automated gate only.** Test + lint/build green **and** a diff-scope check (only the declared `file:line`s changed). No LLM review. For Low-risk pure `Fix` where the safety net is authoritative: dead-code/comment deletion, dependency bumps, orphan CSS/DOM removal. *R0's safety depends on a real net* — a green test that actually exercises the changed code. A dead-delete with zero coverage is R2, not R0, because nothing proves you didn't remove something load-bearing.
- **R1 — cheap-model review.** A same/cheaper-tier `requesting-code-review` on the diff. For Low-risk `Fix` that touches live code but has a passing characterization test — a second pair of eyes catches an accidental over-delete without paying strong-model rates.
- **R2 — strong-model review.** The strongest tier, mandatory, isolated PR. For **any** `Preserve-invariant`, Med/High risk, security/access/data-integrity/rules/cost surface, no-safety-net wave, or an escalation. This is where subtle behavior breaks hide, so the strong model earns its cost.

**Escalation is the cheap safety valve:** any R0/R1 wave jumps to R2 the moment a check fails, the diff strays outside its declared scope, or a lower-tier review flags something. Failure already means STOP, so escalating costs nothing extra.

**Assign the tier at design time** (Step 2), written into the checklist (`- [ ] Wave N (Rx): …`), so execution never improvises and you can eyeball the whole plan's review budget before running a wave.

**Decouple review cadence from commit cadence.** Commits stay one-wave-one-commit (each revertible), but a run of consecutive R0/R1 waves can share a **single** review over the cumulative diff instead of one review per wave — a large hygiene track then costs a couple of reviews, not dozens.

Any `High`-risk wave → strongest execution tier + R2, isolated PR, never batched.
