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
- ⛔ **Sweep the axis, don't patch the instance — this is mechanical, not a reminder.** The reviewer reports a *sample*; the same mistake almost always exists elsewhere under a different wording. **After each fix, immediately `grep` the whole file/module for every term you just touched** (the wrong claim, the corrected claim, the number, the identifier) and check each hit — *before* moving to the next finding. Sweep **both directions**: if you demoted an incorrect ✅, also look for the ⚠️ that should have been ✅.
  - Cheapest place to run it is *while the reviewer is still generating* — you already know what you changed last round.
  - Evidence this is load-bearing: on one 2026-08-06 doc review, **four consecutive rounds'** top blocker was a missed sweep of the *previous* round's fix — one wrong definition silently propagated into a third location. The round that adopted the mechanical grep caught 2 of 7 findings before the reviewer did, one of which the reviewer never found at all.
  - A finding whose fix you cannot sweep (no greppable term) is a signal the claim is vague — restate it as something checkable first.
- Fixes accumulate in the working tree — there is **no per-round commit/push** (there is no PR to push to). Commit once via `/agent-sdlc:commit-message` after the loop converges; hand the PR off via `/agent-sdlc:pr-prepare`.

### 5. Score the round, then converge with /loop

- Re-run this skill via `/loop` until the reviewer returns **no new `blocker` / `warn`** on the current diff. A fully clean pass is the ideal stop.
  - In practice a non-deterministic reviewer rarely hands you one, so the working stop condition is **two consecutive rounds with zero *native* accepted findings** — scored per the scorecard below.
- Each round, log to mem-tmp what was **fixed** and what was **skipped (with the reason)** — this is the audit trail (appleboy keeps it in GitHub PR threads; we keep it in mem-tmp).

#### ⛔ Every round ends with a scorecard — this is not optional

> 🧪 **PROVISIONAL — this whole subsection is an experiment.** Adopted 2026-08-06 at the
> user's request to make "is this loop actually worth its cost" answerable instead of a
> feeling. **It stays until the user says the question is settled, then it comes out.**
> Do not let an experiment ossify into permanent ceremony: if you are filling the table in
> without it changing any decision, say so and propose removing it.

Without numbers you cannot tell "the reviewer is still finding things I missed" from
"the reviewer is now editing my prose", and the loop degenerates into *adding rounds*
instead of *fixing the pipeline*. Report it unprompted. It is **artifact-agnostic** —
the same table works for code, docs, configs, and design reviews.

| Metric | How | Why it's there |
| --- | --- | --- |
| Cost — payload | chars (or tokens) × lanes | raw |
| Cost — wall clock | per-lane harvest seconds | raw |
| Findings / accepted / rejected | per-finding verdict | hit quality |
| **Native accepted** | accepted **and not text you wrote in a previous round** — cite where the text came from | separates "found what I missed" from "co-editing my draft" |
| 🔴 **Escaped defects** | valid findings a **later** round or a human caught that an earlier round should have | **the performance measure** — scored retroactively, not by self-assessment |
| Mechanizable share | how many accepted findings a deterministic check could have produced | tells you what to move out of the sampler |
| Instances you swept up yourself | extra sites found per finding | measures the sampling gap |

##### ⛔ Report the numbers side by side — never collapse them into one score

- **Do not designate a headline metric, and do not divide cost by findings.** A single
  ratio is the thing that gets optimized instead of the work. The pair `(cost, escaped
  defects)` is the goal: **let fewer defects escape, for less** — *not* "maximize findings
  per token".
- ⛔ **Know how you would cheat, because you will otherwise do it accidentally.** The
  earlier version of this section made "cost ÷ native accepted" the headline. That number
  is gameable at both ends, and the failure modes are concrete:
  - **Classification drift** — call a borderline finding "native" and the score improves.
    *Mitigation: `native` requires evidence — name the round or commit that introduced the text.*
  - **Splitting** — report one defect as three findings and the count triples.
  - **Shrinking the payload** — send less, cost falls, coverage falls silently with it.
  - 🔴 **Stopping early** — a cost-per-find metric rewards quitting exactly when finds get
    expensive, and **late expensive finds are the valuable ones**. Measured 2026-08-06: the
    most expensive round of a 10-round run produced that run's single largest correction.
    *An efficiency metric that would have told you to stop before it is a broken metric.*
- **Acceptance rate alone lies.** A round can accept every finding and still be worthless
  if all of it is about your own previous round's wording. Judge over **two** rounds, not one
  (see "Reading N results").
- **After scoring, name one concrete change for the next round** — a changed lens, a check
  moved into a script, a smaller payload. Not a reflection; something that is different next time.
- **Feed what you learn back into this skill.** The scorecard exists to change the pipeline,
  not to decorate the log.

#### Push determinism upstream before spending the sampler

The reviewer **samples**; a checker **enumerates**. Anything a deterministic tool can decide
should never reach the reviewer:

- **Code**: run the type checker, linter, and tests *before* sending the diff. They are free
  and exhaustive; paying an LLM to rediscover what `ruff`/`tsc`/`pytest` already flag is waste.
- **Other artifacts** (docs, configs, schemas): if no checker exists, **that gap is itself a
  finding**. A ~50-line script covering the claim shapes you actually observed beats another round.
- Measured on a 2026-08-06 doc review: of 8 findings in one round, **5 were deterministically
  computable** (table sums, whether cited `file:line` exists and says what's claimed, whether a
  "shared, one place to fix" claim really has one site, cross-table verdict consistency).
  That round sent **125K chars** to get **5 native findings** — and a script produces that class
  of finding in seconds, *exhaustively*, instead of one sampled instance at a time.
  (Note the shape of the win: not "cheaper per finding" but **all instances instead of one**.
  On that round the reviewer named 1 site of a defect that existed in 5.)
- Every accepted finding should leave behind a **re-runnable assertion** — a test for code, a
  check rule for docs. Then "sweep the axis" means *re-run the suite*, not *grep a string*.

#### Round cap

- **Default cap: 10 rounds**, then hand to the user for human review.
- ⚠️ **The cap is a budget, not a verdict.** The original rationale ("past 10 means an
  architectural disagreement, not a fixable finding") was **measured false at least once**:
  on a 2026-08-06 doc review the 10th round had the *highest* acceptance rate of the run
  (7/8) and produced the single largest correction in the document — a load-bearing figure
  that was wrong by 6× and had survived nine prior rounds. **5 of its 8 findings were original
  defects, not the reviewer editing recent edits.**
- **So decide by the scorecard, not the counter**: while native accepted is still non-zero,
  more rounds are justified — say so, show the cost, and let the user choose. ⛔ **Rising cost
  per round is not a reason to stop** (that is the early-stopping trap above); only the
  findings drying up is. The stop condition is **two consecutive rounds with zero native
  accepted findings** (see "Reading N results" for why one round is not enough).

## Multi-lens fan-out (scaling one round up)

The reviewer is **non-deterministic**: send the same input twice and the findings barely
overlap. So a round's silence is *not* evidence of cleanliness — it's one sample. Running N
rounds serially costs N× the wall-clock for the same coverage as N samples drawn at once.

**Fan-out replaces the sampling inside a round; it does not replace the round loop.**
Steps 3 (verify), 4 (fix), and 5 (converge) are unchanged — a fan-out round still ends with
**no commit**, and convergence is still "a full round returns no new `blocker`/`warn`".

### Design the lenses, don't clone the prompt

N copies of the same prompt mostly re-find the same things. Give each lens a **different
question** and forbid it from answering the others. Lenses that worked on a 24K-char audit doc:

| Lens | Asks only | Forced evidence |
| --- | --- | --- |
| **Numbers** | do subtotals, denominators, and "N items" claims add up | the arithmetic that fails |
| **Contradictions** | do two passages assert incompatible things | **quote both passages** |
| **Self-rule violations** | does the doc break a rule it set itself | **quote the rule, then the violation** |
| **Structure / cross-refs** | do headings match content, do "see section X" refs resolve | the ref and its actual target |

Add a global "⛔ out of scope" block to every lens (facts it cannot check, judgments it was
not given the source material for) — otherwise lenses drift into speculating about the domain.

### Delegating N at once

Mechanics belong to the reviewer implementation, not here — for v1 see `sub-gemini` **mode D**.
The one rule that bites regardless of implementation: **parallel send is fine, harvest must be
serial**, because a backgrounded browser tab never renders the streamed response. Its failure is
**silent and looks exactly like "still thinking"**.

### Reading N results — count is not credibility

- ⛔ **Agreement across lenses is not a truth signal.** How many lenses raised a finding measures
  *how easy it was to think of*, not *how likely it is to be true*. Measured: four lenses
  independently asserted the same wrong claim (all four had skipped checking how the code
  handled the upstream value); the single most valuable finding of the round came from **one**
  lens alone. **Verify every finding yourself (step 3) regardless of how many lenses raised it.**
- ✅ **What agreement *is* good for: locating ambiguity in your own writing.** When two
  independent lenses (or two rounds) misread the *same* passage the *same* way, the passage is
  genuinely unclear even if their conclusion is wrong. Reject the claim **and** fix the wording
  — otherwise round N+1 rediscovers it.
- **Record rejections with reasons**, per step 3. Rejected findings recur across rounds; without
  a written rationale you re-litigate each one every time.
- **Stop on the *nature* of findings, not the count** — but read it over **two consecutive
  rounds**, never one. When the reviewer stops finding what you missed and starts co-editing
  your draft, that's converged even if the count hasn't dropped. The metric is the scorecard's
  **native accepted** column (step 5).
  - ⛔ **One round is one sample — it is not enough to stop on.** Measured 2026-08-06: round 8
    produced 2 findings, **both** about the previous round's own edits — by a single-round rule
    that is "converged, stop". Rounds 9 and 10 then produced **9 more accepted findings, most of
    them original defects**, including the run's largest correction. Stopping at 8 would have
    shipped it.
  - So: stop after **two consecutive rounds with zero native accepted findings**.

### Tab hygiene

**Close each reviewer tab as soon as you have harvested it** — it is part of "get the result",
not cleanup for later. Log the conversation URL if you may want to revisit; do not keep tabs
open as bookmarks.

## Relationship to the rest of the pack

- Usually run after `/agent-sdlc:pr-prepare`; update the PR after fixes converge.
- **Not a replacement for human review.** A change `classify-change` rules **core** still needs a line-by-line human pass — this skill is an assistant to that review, not a stand-in for it.

## Context checkpoint

A long review loop is token-heavy. At the end of each round, check context usage; if it is high, update the mem-tmp resume block first, then suggest the user run `/compact`.

## After this gate

<!-- DELIBERATE DELTA vs upstream appleboy/skills: sdlc callback is CONDITIONAL on a progress file existing (standalone single-skill use isn't forced into the full lifecycle). Intentional — don't restore the unconditional "REQUIRED" form. -->

**Running the full agent-sdlc lifecycle?** If a `docs/sdlc/<feature>-sdlc-progress.md` exists (or you deliberately started the whole SOP chain), invoke `/agent-sdlc:sdlc` — it ticks this gate and reports the exact ⏭ next step. It navigates only; it will not run the next gate.

**Used this skill standalone?** You're done — do NOT invoke `/agent-sdlc:sdlc` (there is no progress file for it to update). If you want to keep going, the step that normally follows is **gate 11 — 🚦 human line-by-line review & merge (no skill; the final human gate)**.
