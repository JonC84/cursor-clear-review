# CLEaR Review — a Cursor Agent Skill

<p align="center"><img src="assets/logo.png" width="128" alt="CLEaR Review logo"></p>

A review-only skill for Cursor that audits code against four pillars — **C**lean, **L**ightweight, **E**fficient, **R**obust — and emits a structured findings report plus a **risk-tiered Plan of Attack**. It never edits code; it produces the plan that a (cheaper) model can then execute safely.

## Why it's different

Most "review" prompts dump a wall of suggestions and leave you to sort out what's safe. CLEaR is built around two hard problems:

- **Safe remediation on cheaper models.** Every finding is tagged with **Severity / Effort / Risk** and a **model tier** (Any / Capable / Strongest+review). The Plan of Attack sequences the work into small, revertible waves so a low-cost model can implement most of it, with a **right-sized diff review** (R0 automated-only · R1 cheap · R2 strong) as the cheap guardrail — so a review never costs more than the work it guards.
- **Not breaking intentional behavior.** Every finding is triaged as **Fix / Preserve-invariant / Verify-intent / Won't-fix**. Load-bearing behavior (auth/offline boot, cache/prefetch correctness, security integrations, telemetry) is flagged to *confirm before touching* — the skill actively resists "defect-elimination bias."

## The four pillars

- **Clean** — dead/duplicated/superseded code; comments (default: none, with an explicit keep-list for the few that encode real intent).
- **Lightweight** — oversized files/functions, repeated boilerplate, needless indirection; focused splits and shared helpers.
- **Efficient** — first-class **backend/usage cost**: unbounded DB reads, N+1s, over-fetching, chatty writes, storage egress, serverless waste, client/network jank. Every efficiency finding states a cost impact.
- **Robust** — swallowed errors, races, offline/auth edge cases, permissive access rules, fragile APIs, input validation.

## Two modes

- **Scoped (default):** review a diff/branch/named files after a feature or before merge.
- **Exhaustive:** a whole-codebase health pass in prioritized batches. When the resulting plan is too big for one session, the skill drives remediation via a master triage plan + per-track plans, executed **one wave per fresh chat** with copy-paste prompts. See [`skills/clear-review/references/exhaustive-remediation.md`](skills/clear-review/references/exhaustive-remediation.md).

## Install

This repo is packaged as a **Cursor plugin** (`.cursor-plugin/plugin.json` + `skills/clear-review/`). You can install it as a plugin or drop the skill in directly.

### A. As a plugin

**Cursor Marketplace (once published):** open **Customize** in the Cursor sidebar, search for `clear-review`, and click Install. Or browse [cursor.com/marketplace](https://cursor.com/marketplace).

**Team marketplace (admins):** Dashboard → Plugins → Team Marketplaces → Add Marketplace → Import from Repo, then paste this repo's GitHub URL. Developers install it from **Customize**.

**Local (development / self-serve):** clone, then symlink into Cursor's local plugins dir and reload the window.
```bash
git clone https://github.com/JonC84/cursor-clear-review
ln -s "$(pwd)/cursor-clear-review" ~/.cursor/plugins/local/clear-review
# Windows PowerShell:
# New-Item -ItemType SymbolicLink -Path "$HOME\.cursor\plugins\local\clear-review" -Target "$PWD\cursor-clear-review"
```
Then restart Cursor or run **Developer: Reload Window**.

### B. As a plain skill (no plugin)

Copy just the skill folder — simplest for a single user or to vendor into a repo.
```bash
# Personal (all your projects):
cp -r cursor-clear-review/skills/clear-review ~/.cursor/skills/clear-review
# Project (shared via a repo):
cp -r cursor-clear-review/skills/clear-review <your-repo>/.cursor/skills/clear-review
```
Windows PowerShell: `Copy-Item -Recurse cursor-clear-review\skills\clear-review $HOME\.cursor\skills\clear-review`

> Note: Cursor does not offer a "paste this URL to install a plugin" one-liner for individuals. Until it's on the public Marketplace (reviewed) or imported into a team marketplace, the self-serve options are the **local symlink** (A) or the **plain-skill copy** (B).

### Use

In Agent chat, ask for a "CLEaR review", or invoke `/clear-review`.

## Pairs with (optional): Superpowers

CLEaR hands off to these Superpowers-plugin skills for the remediation phase: `brainstorming`, `test-driven-development`, `requesting-code-review`, `verification-before-completion`, `writing-plans`. If you don't have Superpowers installed, the handoffs simply become "use your own equivalent step" — the review contract and Plan of Attack are fully self-contained.

## What a report looks like

```
## CLEaR Review Report
**Scope:** Scoped — feature/x (3 files)
**Summary:** <=3 sentences.

### Findings
- **E1** `db.js:292` — Severity: Critical · Effort: M · Risk: Med · Class: Preserve-invariant
  - Finding: getAllDiagrams() reads the whole collection every call.
  - Recommendation: server-side scoped/paginated search.
  - Invariant: modal must still surface the same diagrams.
  - Cost impact: Firestore reads — O(N) per open.
...

### Plan of Attack
- Wave 1 — <name>: IDs · Tier: Capable · Review: R1 · Safety net: <tests> · Verify: <checks>
...
### Next step
Review complete — no code changed. Handing off to remediation design.
```

## Changelog

- **1.1.0** — Per-wave **Review tiers** (R0/R1/R2); fresh-chat handoffs; **track-completion continuity**; **Automated mode** at the spec/plan handoff with **Continuous** or **Pause** (new chat optional for Automated). Pause prints **Quick manual checks**. Progressive Standard-step ticks; **Wave N is checked only after steps 1–7 are complete**.
- **1.0.0** — Initial release.

## License

MIT — see [LICENSE](LICENSE).
