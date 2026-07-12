---
name: sdlc
description: Update the current feature's SDLC progress file and report the next step. Use when the user asks "what's next", "where am I in the SDLC", "update the SDLC progress", "I finished <a gate>", or when another agent-sdlc skill calls back after finishing its gate. It reads the per-feature progress file plus git state, ticks completed gates, refreshes the "current → next step" block, and reports 📍current / ⏭next / blocker. It NAVIGATES only — it never runs the next gate, never approves a human gate, and never touches version control.
---

# SDLC Navigator — progress-file updater

You maintain **one progress file per feature** and tell the user what to do next. You are a **navigator, not a driver**: you read state, update the file, and report the next step. You never execute the next gate, never sign off a human gate, and never commit.

The full 12-step chain is the pack's `SOP.md`. The progress file is the single source of truth for *one* feature's journey through that chain.

## When you run

- The user explicitly invokes `/agent-sdlc:sdlc`.
- Another `agent-sdlc` skill finished its gate and called back (its closing "After this gate" step).
- The user says things like "what's next", "update progress", "I just finished code-review".

## Step 1 — Locate the progress file

- Look in the current dev repo for `docs/sdlc/<feature>-sdlc-progress.md`.
- If several exist, pick the one whose feature/branch matches the current work; if ambiguous, ask the user which feature.
- If none exists, go to **Initialization** below.

## Step 2 — Read current state

- Read the progress file.
- Read git state to corroborate: current branch, staged/unstaged changes, recent `git log --oneline -n 5`.
- Note what the current conversation has just done — this is how you reconcile the **built-in** gates (5 `/simplify`, 6 `/security-review`, 7 `/code-review max -fix`), which are Claude Code built-ins that cannot call back on their own. When you are invoked as the callback right after gate 8 (commit-message), tick 5/6/7 too if they were run in this session.

## Step 3 — Update the file

- Tick completed gates: `- [ ]` → `- [x]`; mark the in-progress one with 🔄.
- Refresh the top block **📍 目前位置 → ⏭ 下一步 → 卡點** — this is the landing spot the user reads to know the next step. Make "下一步" a concrete action (e.g. `` `/simplify` ``, `/agent-sdlc:commit-message`, "human review the core diff").
- Append known facts to **紀錄 / 連結**: commit hashes, PR URL, external-review summary, and any **decision/drift** (e.g. planned leaf but the real diff drifted core — record the upgrade).
- Do **not** invent completion. If git state does not corroborate a gate, leave it unticked and say so.

## Step 4 — Report to the user

Report three lines:
- **📍 目前**: which gate the feature is at.
- **⏭ 下一步**: the exact next action/command.
- **卡點**: none, or the blocker.

If the user tried to skip a gate (e.g. wants to merge with gate 10 external-ai-review unticked), point it out plainly — remind, don't block (the pack does not self-simplify gates, but you are a navigator, not a gatekeeper).

## Initialization (no progress file yet)

1. Copy the bundled template into the dev repo: from this plugin's `templates/sdlc-progress.template.md` (one level up from `skills/`, i.e. `../../templates/sdlc-progress.template.md`) → `docs/sdlc/<feature>-sdlc-progress.md`.
2. Fill the header: feature name, open date, planning tool (superpowers | plan-feature), base/work branch, and the leaf/core verdict if known.
3. Ensure `docs/sdlc/` is gitignored in the dev repo (it is a working scratch, same nature as mem-tmp — it must never enter version control). If the repo's `.gitignore` does not cover it, tell the user to add `docs/sdlc/`.
4. Then run Steps 2–4.

## Close-out (feature merged — gate 11 done)

- Confirm gate 11 (human review & merge) is complete and the PR is merged.
- Report "此功能 SDLC 完成".
- **Delete the progress file.** Do not archive it — the commit(s) and PR are the permanent record. There is no archive step.

## Hard boundaries

- ❌ Do **not** run the next gate (do not invoke `/simplify`, do not commit, do not open a PR). You only *say* what the next step is.
- ❌ Do **not** approve any 🚦 human gate — approval is always the human's.
- ❌ Do **not** `git add` / `git commit` the progress file, and do not commit anything else. The progress file is gitignored scratch.
- ❌ Do **not** edit the SDLC gate logic of the other skills.

## Relationship to the rest of the pack

- The 6 pack skills (gates 0/1/3/8/9/10) each call back to you when their gate completes — that is what keeps the progress file current without the user thinking about it.
- Built-in gates 5/6/7 reconcile at the next pack gate (8) — the progress file carries that truth across any `/compact`.
