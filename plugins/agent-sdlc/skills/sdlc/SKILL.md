---
name: sdlc
description: Update the current PR's SDLC progress file and report the next step. Use when the user asks "what's next", "where am I in the SDLC", "update the SDLC progress", "I finished <a gate>", or when another agent-sdlc skill calls back after finishing its gate. It reads the per-PR progress file plus git state, ticks completed gates, refreshes the "current → next step" block, and reports 📍current / ⏭next / blocker. It NAVIGATES only — it never runs the next gate, never approves a human gate, and never touches version control.
---

# SDLC Navigator — progress-file updater

You maintain **one progress file per PR** and tell the user what to do next. You are a **navigator, not a driver**: you read state, update the file, and report the next step. You never execute the next gate, never sign off a human gate, and never commit.

The full 12-step chain is the pack's `SOP.md`. The progress file is the single source of truth for *one PR's* journey through that chain.

⛔ **One PR, one file — never one file per multi-PR feature.** You read this file at *every*
gate, so a shared one grows without bound: measured on a 4-PR feature, **845 lines / 80 KB**,
still carrying the gate-by-gate detail of a PR that had merged days earlier. Anything the next
PR genuinely needs (the feature-level gate 0 output, cross-PR decisions, still-open items)
belongs in the **plan**; the progress file points back to it.

## When you run

- The user explicitly invokes `/agent-sdlc:sdlc`.
- Another `agent-sdlc` skill finished its gate and called back (its closing "After this gate" step).
- The user says things like "what's next", "update progress", "I just finished code-review".

## Step 1 — Locate the progress file

- Look in the current dev repo for `docs/sdlc/<feature>-<pr>-sdlc-progress.md`.
- If several exist, pick the one whose PR/branch matches the current work; if ambiguous, ask the user which PR.
- If none exists, go to **Initialization** below.

## Step 2 — Read current state

- Read the progress file.
- Read git state to corroborate: current branch, staged/unstaged changes, recent `git log --oneline -n 5`.
- Note what the current conversation has just done — this is how you reconcile the **built-in** gates (5 `/simplify`, 6 `/security-review`, 7 `/code-review max -fix`), which are Claude Code built-ins that cannot call back on their own. Tick 5/6/7 at the **first pack gate that follows them** — in the pack's execution order `… → 7 → 9 → 10 → Δ → 8 → 11` that is gate 9 (`pr-prepare`), not gate 8.
- ⚠️ **Read the order from `SOP.md`, not from the numbers.** Gate 8 (commit) runs after 9, 10 and Δ. A feature sitting at gate 7 has ⏭ `/agent-sdlc:pr-prepare` next, not commit-message.

## Step 3 — Update the file

- Tick completed gates: `- [ ]` → `- [x]`; mark the in-progress one with 🔄.
- Refresh the top block **📍 目前位置 → ⏭ 下一步 → 卡點** — this is the landing spot the user reads to know the next step. Make "下一步" a concrete action (e.g. `` `/simplify` ``, `/agent-sdlc:commit-message`, "human review the core diff").
- Append known facts to **紀錄 / 連結**: commit hashes, PR URL, external-review summary, and any **decision/drift** (e.g. planned leaf but the real diff drifted core — record the upgrade).
- Do **not** invent completion. If git state does not corroborate a gate, leave it unticked and say so.
- **Gate Δ (`/agent-sdlc:delta-enumerate`) needs its own line, not just a tick**: which lines were the population and how it was derived, what was enumerated, how many findings, whether the severity threshold was met (≥1 finding that reaches a dangerous path or overturns a written guarantee → escalate to a second full gate 7), **what Δ fixed vs what it left for human review as two separate lists**, and the Δ′ result. If gates 9/10 accepted nothing, Δ's population is empty — record "Δ skipped, empty population". ⚠️ "Δ ran and found nothing" and "Δ never ran" must not look the same on the file.
- **Gates 5/6 are provisional until gate 7 clears them.** When ticking 5 or 6, record their findings as *pending gate 7* — do not let them acquire decision numbers or enter the plan's decision section yet. Measured: gate 7 has overturned a reported gate-5 conclusion and a reported gate-6 conclusion, both "treating one measurement as the spec".

## Step 4 — Report to the user

Report three lines:
- **📍 目前**: which gate the feature is at.
- **⏭ 下一步**: the exact next action/command.
- **卡點**: none, or the blocker.

If the user tried to skip a gate (e.g. wants to merge with gate 10 external-ai-review unticked), point it out plainly — remind, don't block (the pack does not self-simplify gates, but you are a navigator, not a gatekeeper).

## Initialization (no progress file yet)

1. Copy the bundled template into the dev repo: from this plugin's `templates/sdlc-progress.template.md` (one level up from `skills/`, i.e. `../../templates/sdlc-progress.template.md`) → `docs/sdlc/<feature>-<pr>-sdlc-progress.md`.
2. Fill the header: feature name, open date, planning track (see below), base/work branch, and the leaf/core verdict if known.

   **Planning track — chain the tools, don't pick one.** They sit at *different altitudes* and compose; they are not alternatives:

   | Stage | Tool | Produces |
   |:--|:--|:--|
   | a. Clarify + compare approaches | `superpowers:brainstorming` *(if installed)* | an approved design/spec; forces 2–3 approaches with trade-offs, asks one question at a time |
   | b. **Decision layer (mandatory)** | `/agent-sdlc:plan-feature` | goal, may-modify + **must-not-touch**, verification strategy, Mermaid, risks, **rollback** |
   | c. Implementation layer | `superpowers:writing-plans` *(if installed, and only when the work is code-heavy enough to need per-task steps)* | per-task red-then-green steps |

   Stage **b is never skippable** — it is the only one that produces the must-not-touch list, the rollback, and the planning-exit checklist items. Stages a and c degrade gracefully: if superpowers is not installed, skip them and note that in the progress file.

   Record which stages actually ran, e.g. `brainstorming + plan-feature`.
3. Ensure `docs/sdlc/` is gitignored in the dev repo (it is a working scratch, same nature as mem-tmp — it must never enter version control). If the repo's `.gitignore` does not cover it, tell the user to add `docs/sdlc/`.
4. Then run Steps 2–4.

## Close-out (this PR merged — gate 11 done)

- Confirm gate 11 (human review & merge) is complete and the PR is merged.
- Report "此 PR 的 SDLC 完成".
- **Move anything still live into the plan first** — open items, decisions that outlive this PR, hard inputs the next PR depends on. Then **delete the progress file.** Do not archive it — the commit(s) and PR are the permanent record. There is no archive step.
- If the feature has further PRs, the next one starts a **new** file; do not reopen this one.

## Hard boundaries

- ❌ Do **not** run the next gate (do not invoke `/simplify`, do not commit, do not open a PR). You only *say* what the next step is.
- ❌ Do **not** approve any 🚦 human gate — approval is always the human's.
- ❌ Do **not** `git add` / `git commit` the progress file, and do not commit anything else. The progress file is gitignored scratch.
- ❌ Do **not** edit the SDLC gate logic of the other skills.

## Relationship to the rest of the pack

- The **7** pack skills (gates 0/1/3/8/9/10/**Δ**) call back to you when their gate completes **and a progress file exists** (i.e. the PR is running the full lifecycle, not a standalone single-skill use) — that is what keeps the progress file current without the user thinking about it.
- Built-in gates 5/6/7 reconcile at the next pack gate — in execution order that is gate **9** (`pr-prepare`), not gate 8. The progress file carries that truth across any `/compact`.
- ⚠️ **Gate 0 is feature-scoped, not PR-scoped.** For the second and later PRs of one feature, `prompt-audit` is not re-run; the progress file records "not re-run, see \<where\>". Do not report it as a skipped gate.
