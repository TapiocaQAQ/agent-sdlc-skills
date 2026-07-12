---
name: external-ai-review
description: Get a second-opinion AI review of the current diff and fix what it legitimately finds. A model-agnostic, pluggable external reviewer (v1 = sub-gemini, driving Gemini through the browser), run as a single-pass check-fix cycle designed to pair with /loop until the diff comes back clean. This is the pack's local, pre-PR, no-extra-cost stand-in for appleboy's copilot-review / codex-review (which need a paid Copilot/Codex subscription and a pushed GitHub PR). Use when the user wants an external or second AI opinion on a change, asks to "review this diff", "second-opinion review", "AI review before the PR", or wants to loop review-then-fix until clean.
---

# External AI Review

Give a *finished, ready-for-PR* change a second pair of eyes from whatever external AI reviewer is available, then fix what it legitimately finds. **The reviewer is an interface, not a hard-wired model** — adding a new reviewer means implementing the interface, not rewriting this flow.

This is the pack's answer to appleboy's `copilot-review` / `codex-review`. Those drive real GitHub PR bots (Copilot / Codex) over a *pushed* PR and manage review threads via `gh`. This skill instead reviews a **local diff** with a **synchronous** reviewer, so it works **before** a PR exists and costs nothing beyond your existing Gemini access. (`copilot-review` / `codex-review` stay on the roadmap as alternative reviewer implementations — see the pack README.)

## When to use

- After `/agent-sdlc:pr-prepare` produces a PR description, or before committing, when you want an independent second review.
- Mirrors appleboy's `/loop 3m /copilot-review`: pair with `/loop` to re-review until the diff is clean.

## Reviewer interface (pluggable)

A reviewer must be able to: ① take a *change summary + diff + focus points*, and ② return a *list of findings* (`file:line` + severity + suggested fix).

- **v1 implementation = sub-gemini** (Gemini Pro, driven through your logged-in Chrome per the `sub-gemini` skill).
- When a Copilot / Codex CLI (or any other reviewer) becomes available later, implement the same interface and it slots in — this flow does not change.

## Flow (single-pass check-fix)

Each invocation runs **one** cycle. Pair with `/loop` to repeat until clean.

### 1. Prepare the review input

- Get the change with `git diff` (or `git diff main...HEAD` for a branch).
- Summarize: what changed, `classify-change`'s leaf/core verdict, and what the reviewer should look hardest at (e.g. money precision, auth, boundaries, error paths).
- ⛔ **Sensitive-data red line — this is the gate that makes an *external* reviewer safe.** Before anything leaves your machine for the reviewer, **strip every piece of real data, secret, or PII**: real CSV/DB contents, invoice-carrier numbers, verification codes, `.env` values, API keys, tokens, real user records. Send **only source-code diff and synthetic examples**. `copilot-review` / `codex-review` never needed this — their diff is already inside GitHub; here the diff is *sent out* to Gemini, so redaction is mandatory. If a hunk cannot be shown without real data, replace the data with a synthetic placeholder or omit that hunk and note it.

### 2. Delegate to the reviewer (v1: sub-gemini)

- Invoke the `sub-gemini` skill and follow its full playbook (open a fresh `/app` tab, fill then send via JS `eval`, verify the send with `get url`, poll until the "stop responding" control disappears, then read the last `message-content`). Do not duplicate those browser details here — the `sub-gemini` skill owns them.
- Make the prompt self-contained: attach the change summary + the **redacted** diff + an explicit ask: "List findings as `file:line`, severity (`blocker` / `warn` / `nit`), and how to fix."

### 3. Verify the reviewer's findings (Claude's job — do NOT blindly accept)

- Gemini hallucinates more than a code-tuned bot, so **every finding is a claim to check, not an instruction to obey.** For each one, confirm against the actual diff that the code and the `file:line` really exist and that the problem is real before accepting it.
- For any finding that touches a fact, version, or API contract, verify it independently — do not take Gemini's word.
- Drop hallucinated, stale, or architecture-conflicting findings, and record *why* each was dropped (this is the local equivalent of appleboy's "reply with a rationale before resolving").

### 4. Fix

- Fix each accepted `blocker` / `warn` in the code, and add a test that pins the fix where it makes sense.
- Handle `nit`s at your discretion.
- Fixes accumulate in the working tree — there is **no per-round commit/push** (there is no PR to push to). Commit once via `/agent-sdlc:commit-message` after the loop converges; hand the PR off via `/agent-sdlc:pr-prepare`.

### 5. Converge with /loop

- Re-run this skill via `/loop` until the reviewer returns **no new `blocker` / `warn`** on the current diff. That clean pass is the only stop condition.
- Each round, log to mem-tmp what was **fixed** and what was **skipped (with the reason)** — this is the audit trail (appleboy keeps it in GitHub PR threads; we keep it in mem-tmp).
- **Cap at 10 rounds.** Past that usually means an architectural disagreement, not a fixable finding — stop and hand it to the user for human review.

## Relationship to the rest of the pack

- Usually run after `/agent-sdlc:pr-prepare`; update the PR after fixes converge.
- **Not a replacement for human review.** A change `classify-change` rules **core** still needs a line-by-line human pass — this skill is an assistant to that review, not a stand-in for it.

## Context checkpoint

A long review loop is token-heavy. At the end of each round, check context usage; if it is high, update the mem-tmp resume block first, then suggest the user run `/compact`.

## After this gate

When this gate is complete, **REQUIRED: invoke `/agent-sdlc:sdlc`** to update the feature's SDLC progress file — it marks this gate done and refreshes the "current → next step" block so you always know what comes next. (If no progress file exists yet, `sdlc` creates one from the pack template. `sdlc` only navigates — it will not run the next gate for you.)
