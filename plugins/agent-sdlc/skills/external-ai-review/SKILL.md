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

- ⛔ **Name the payload's edges, and send the *rendered* artifact, not only the rule that produces it.** A reviewer cannot tell "this file is absent because it is out of scope" from "this reference is dead", and it cannot judge whether an edit reads well when all it has is the edit *table*. Measured on a 2026-08-07 round: **3 of 7 rejected findings were caused by the input, not by the reviewer** — two lens findings claimed a description had lost its entry condition (the surviving text said it in the first two sentences, which were not in the payload), and one reported a live cross-reference as a dead link (both files are version-controlled, just not in the diff).
  - So: list the files that are referenced but deliberately excluded, and say they exist. Say which artifacts are generated.
  - And when the change is a *transform* (a rewrite table, a codegen rule, a migration), attach **before/after of the actual output** for the affected items. Measured cost on that round: +29,230 chars, +41% on that one lane and +10% on the round — cheap next to the rejections it removes. This is a targeting change, not truncation: nothing is taken away.

### 2. Delegate to the reviewer (v1: sub-gemini)

- Invoke the `sub-gemini` skill and follow its full playbook (open a fresh `/app` tab, fill then send via JS `eval`, verify the send with `get url`, poll until the "stop responding" control disappears, then read the last `message-content`). Do not duplicate those browser details here — the `sub-gemini` skill owns them.
- Make the prompt self-contained: attach the change summary + the **redacted** diff + an explicit ask: "List findings as `file:line`, **how you would falsify this claim from the material you were given**, and how to fix."
- ⛔ **Ask for a falsifier, not a severity.** Severity is the reviewer grading its own homework, and it grades generously: measured across two fan-out rounds of one code review, **13 findings came back labelled `blocker`; 9 were rejected or downgraded on inspection, and exactly 1 was a defect that reached runtime behaviour.** One lens returned 5 blockers of which 0 survived. A severity field costs nothing to inflate and tells you nothing you would not learn in step 3 anyway.
  - What a **falsifier** buys you: the finding arrives with the experiment attached. Replayed against that round, every rejected blocker was killed by a measurement *I had to design myself* (does any schema key contain a blocked name? does `$ref` appear anywhere? what does anyio actually put in the group?). Demanding the recipe up front moves that work to the reviewer without losing a single finding.
  - Keep severity if you like it, but **read it as the reviewer's confidence, never as yours** — and never let it order your work.
  - ⚠️ Its own failure mode: a reviewer can invent a plausible-sounding falsifier the same way it invents a severity. You still choose and run the measurement; the recipe is a hint, not a result.
  - ⛔ **Run the falsifier, then check the premise it takes for granted.** A falsifier tests
    the *claim*; it cannot test the *requirement the claim invokes*. Measured 2026-08-07: a
    finding said a probe's `samples_ms` must be sorted "to satisfy the cross-run comparison
    requirement", and supplied a falsifier that ran clean — the array is indeed unsorted. The
    finding was still wrong, because **no such requirement existed**: nothing compares that
    array between runs, and sorting would have destroyed the only within-run drift signal it
    carries. Ask of every finding: *what does this assume is required, and is it?*
  - ✅ Where the field actually paid: two findings that round were killed by a **single**
    measurement, and reading their falsifiers is what told me which one measurement to run.
    Their own recipes were useless (one grepped a variable name the reviewer had invented),
    but they revealed the shared assumption — so I falsified the assumption instead of the
    two claims. **The recipe's value is often the assumption it exposes, not the check it names.**

### 3. Verify the reviewer's findings (Claude's job — do NOT blindly accept)

- Gemini hallucinates more than a code-tuned bot, so **every finding is a claim to check, not an instruction to obey.** For each one, confirm against the actual diff that the code and the `file:line` really exist and that the problem is real before accepting it.
- For any finding that touches a fact, version, or API contract, verify it independently — do not take Gemini's word.
- Drop hallucinated, stale, or architecture-conflicting findings, and record *why* each was dropped (this is the local equivalent of appleboy's "reply with a rationale before resolving").
- ⛔ **A finding names an axis, not a list. Sweep that axis yourself in both directions before closing it.**
  The reviewer sampled; it did not enumerate. Verifying the one instance it happened to see and
  stopping there banks the sample as if it were the population.
  Measured on the 2026-08-09 PR-A review: the finding was *"the doc's coverage claim skips
  `input_schema`"* — on its face one over-stated sentence. Sweeping that axis (what does the
  downstream gate **actually** scan, and had we run **all** of its checks?) turned up that the audit
  had only ever run one of the gate's **two** prose checks; the other one hit two more tools and moved
  a security-bearing lower bound from 5 to 7. **That was not in the reviewer's reply**, and following
  step 3 literally — confirm the `file:line`, fix the sentence — would have shipped without it.
  → For each accepted finding, ask: *what is the general form of this mistake, and where else could it
  be?* Enumerate that set; do not pattern-match a second instance.
- ⛔ **Judge the observation and the proposed fix separately — they fail independently.**
  A finding can be *right about the defect* and *wrong about the remedy*, and accepting it
  wholesale ships the remedy. Measured twice on one 2026-08-07 review:
  - A reviewer correctly found a scan that missed a spelling, and proposed normalising
    spaces. Its own measurement looked free — **0 false positives on all 35 recorded
    inputs**. But that scan fails *startup*, and against plausible English it fired on 6 of
    6: "Returns host health and uptime", "A cluster health summary is not available here" —
    the second being a sentence the project would itself write. The fix would have shipped
    a container that will not boot.
  - Another correctly found an unguarded reference, and proposed overloading an unrelated
    table to hold it, which would have broken that table's own invariant.
  In both cases the right move was **accept the observation, redesign the fix** — and, where
  the boundary is deliberate, *write the limit down and pin it with a test* so the next
  round does not re-propose the dangerous version. ⚠️ Measuring the fix against the data you
  *have* is not enough when the fix guards against data you do not have yet; test it against
  what could plausibly arrive.

### 4. Fix

- Fix each accepted `blocker` / `warn` in the code, and add a test that pins the fix where it makes sense.
- Handle `nit`s at your discretion.
- ⛔ **Sweep the axis, don't patch the instance — this is mechanical, not a reminder.** The reviewer reports a *sample*; the same mistake almost always exists elsewhere under a different wording. **After each fix, immediately `grep` the whole file/module for every term you just touched** (the wrong claim, the corrected claim, the number, the identifier) and check each hit — *before* moving to the next finding. Sweep **both directions**: if you demoted an incorrect ✅, also look for the ⚠️ that should have been ✅.
  - ⛔ **It has to produce a table, or it does not happen.** Written as a reminder it gets skipped — measured: on a 2026-08-07 code review the sweep was in this skill the whole time and **three consecutive rounds' escaped defects were all sibling sites of the previous round's own fix** (a claim of "all three branches" that covered two; two of three identical matchers case-folded and the third left alone; a count that was wrong the moment it was written). So make it an artifact, landed in mem-tmp before the next round starts:

    | term I changed | grep hits | verdict per hit |
    | --- | --- | --- |

    The terms are every identifier, number, and phrase you touched — plus the corrected form, so you sweep **both directions**. The round that first produced this table found **7 sites, 6 of which the reviewer never mentioned at all**.
  - ⛔ **Any claim of the form "X, because \<arithmetic\>" gets its own row, and the verdict
    column must show the recomputation — not a yes/no.** Counts you can grep; formulas you
    have to evaluate, and a formula in confident prose reads exactly like a verified one.
    Measured 2026-08-07: a probe docstring said raising the sample count from 5 to 10 stopped
    the nearest-rank p95 from being the maximum. `ceil(0.95 × 10) = 10` — index 10 of 10, still
    the maximum. **The recorded data was its own disproof** (p95 == max on every row), and the
    claim survived *five* review rounds and a max-effort code review because every pass read it
    as a measurement instead of evaluating it. Sweeping that one claim then found a second site
    the reviewer never mentioned. Include: percentile/index arithmetic, "N of M", percentages,
    thresholds, complexity claims, and any "so that means" that connects two numbers.
  - Cheapest place to run it is *while the reviewer is still generating* — you already know what you changed last round.
  - ⚠️ Read a table of all-zero hits as **"I aimed it wrong"**, not as "the code is clean". And it only catches *greppable* siblings: the same false claim rephrased is invisible to it. That is a bound on the method, not a reason to skip it.
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
  moved into a script, attention aimed at a region. Not a reflection; something that is
  different next time. ⛔ **Then run it through the four-test gate below before you say it.**
  "A smaller payload" is *not* a change you may propose on its own — it must arrive as the
  consequence of mechanizing an axis (test ③), never as the goal.
- **Feed what you learn back into this skill.** The scorecard exists to change the pipeline,
  not to decorate the log.

#### ⛔ Carry a cross-round coverage ledger — an unexamined region is not a clean one

The scorecard tells you whether the *reviewer* is still productive. It cannot tell you which
parts of the artifact have **never been looked at** — and that is where the oldest defects sit,
because they were written before round 1 and no lens has been pointed at them since.

⚠️ **The step-4 sweep cannot cover this, by construction.** That sweep is anchored on *terms
you just changed*, so text written once and never edited is permanently outside it. The two
mechanisms are complementary: the sweep finds siblings of *this round's* edits; the ledger
finds regions *no round* has questioned. Measured 2026-08-07 on a 5-round code review: the 9
lenses of rounds 1–3 asked exclusively about the three modules under active edit. The build
files and the measurement probes were first examined in round 4, and the run's oldest defect —
a percentile claim written at gate 4, wrong on evaluation, which had survived five review
rounds *and* a max-effort code review — did not surface until round 5.

Carry one table forward across rounds, and let it order the next round's lenses:

| region | round · lens | the question that lens asked |
| --- | --- | --- |

- **A region with no rows gets a lens next round**, ahead of a third pass at the hot module.
  Purely additive — it changes *aim*, not payload (see "Targeting is not truncation").
- ⛔ **The "question asked" column is the load-bearing one — a bare file list is worse than no
  table at all.** Coverage without the question records *someone looked*, and a later round
  reads that as *it is clean*. Measured on that same run: round 4's build lens **did** read the
  probe holding the percentile defect, and returned a finding about that very file, which was
  rejected as an invented requirement. A file-only ledger would have marked the probe *covered,
  nothing found* and deprioritized it. The defect surfaced only because round 5 asked that
  region a **different** question — not because it got a second look.
- ⚠️ Its own failure mode: the ledger records only questions you thought to ask, so an axis
  nobody ever conceived reads exactly like a well-covered one. It bounds **repetition**, not
  **blindness**. A full table means "stop re-asking", never "everything has been checked".

#### ⛔ Gate every proposed change through these four tests — *before* you say it out loud

A pipeline change is a claim about the future. Run these yourself; do not make the user
find the hole. All four are artifact-agnostic — they work for code, docs, configs, schemas.

**① Replay it against the round you just finished.**
For each finding accepted this round, name the evidence it actually needed, then check that
evidence still reaches the reviewer under the proposed pipeline. **A change that would have
missed a finding the current pipeline just produced is a regression, whatever it saves.**
This is the performance test — it is concrete, replayable, and cannot be argued with.

> Measured 2026-08-06: "review only the diff since last round" was proposed and killed by
> its own replay. The round's best finding needed the *changed* line (`43 支`, in the diff)
> **and** the *unchanged* rule that made it wrong (`投影母體是 42`, not in the diff).
> **Defects live in the new text; the criteria that make them defects live in the old text.**
> Scoping the payload by "where the defects are" is a category error.

**② Measure every quantitative claim before speaking it.**
Run `wc`/`du`/the timer first. Same session: "payload drops 140K → 15K" was asserted, never
measured; the honest diff was 29,898 vs 33,988 chars — **12%, not 90%**. An unmeasured number
in a proposal is a guess wearing a lab coat.

**③ Removing coverage requires a mechanization proof, never a score.**
You may retire or repurpose a lane **only** when a deterministic checker now covers its axis
*and* that checker has passed a negative control. ⛔ "That lane scored zero" is not a reason —
zero is equally consistent with *the lane is weak* or *this was one quiet sample*, and cutting
on it is the shrink-the-payload gaming vector wearing a justification.

**④ State the failure mode of your own proposal.**
If you cannot say what would make it wrong, you have not thought about it yet — present the
uncertainty instead of the recommendation. And separate what your change *caused* from what
merely *co-occurred*: one round cannot distinguish "my change worked" from "the artifact ran
out of original defects". Say which you can attribute and which you cannot.

**Targeting is not truncation.** Pointing the reviewer at the changed region while still
sending the whole artifact is a targeting change (safe under ①). Deleting the rest is a
truncation change (usually fails ①). Know which one you are proposing.

#### Push determinism upstream before spending the sampler

The reviewer **samples**; a checker **enumerates**. Anything a deterministic tool can decide
should never reach the reviewer:

- **Code**: run the type checker, linter, and tests *before* sending the diff. They are free
  and exhaustive; paying an LLM to rediscover what `ruff`/`tsc`/`pytest` already flag is waste.
- ⛔ **"Tool X reports clean" is not a claim until the tool's version *and* rule set are
  pinned.** Otherwise it silently changes meaning between rounds and you will read a tool
  upgrade as a regression in your own code. Measured 2026-08-07: `uv tool run ruff` resolves
  to latest, ruff 0.16.1 widened its default rule set, and **unchanged code that reported
  clean in round 3 reported 21 errors in round 4** — while the round-3 rule set still passed.
  Pin it, then record the call on each newly-surfaced rule (adopted / fixed / declined-with-
  reason) so the next person does not re-derive it. The same trap applies to any
  auto-updating checker: formatters, `npm audit`, type checkers, CI base images.
  - This belongs to the round's sweep, not just its setup: your sweep covers claims *you*
    wrote, and this is a claim your *tooling* wrote on your behalf.
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

> ⛔ **The next gate is Δ — named here on purpose, not left to the navigator alone.**
> Δ is the one gate with no skill of its own, so nothing calls back for it; if you skip the
> navigator, nothing else will say the word "Δ". **Hand-editing the progress file is not a
> substitute for invoking the navigator** — the file is the navigator's *output*, and writing
> the output yourself means the ⏭ line is whatever you remembered, not what the chain derives.
> Measured 2026-08-09: exactly that happened, and Δ came out written as "Δ (if needed)".
> **Δ is skippable only when gates 9/10 adopted *zero* findings** (empty population). Adopting
> even one makes it mandatory — do not soften that into "if needed".

**Used this skill standalone?** You're done — do NOT invoke `/agent-sdlc:sdlc` (there is no progress file for it to update). If you want to keep going, the step that normally follows is **gate Δ — enumerate every line this round's gates changed, fix the sub-threshold findings in place, then run one bounded confirmation pass (Δ′) over just those fix lines; escalate anything above threshold to a second gate 7. Defined in the pack's `SOP.md`** — then gate 8 `/agent-sdlc:commit-message`, then gate 11.

⚠️ **Δ's population is the code *this skill just changed*.** Every fix applied in step 4 was written by someone who had, moments earlier, been shown to be thinking too narrowly about that exact spot — which is the one place another round of *sampling* structurally cannot see, because it looks already-fixed. A clean final round means "the reviewer found nothing new by sampling", never "those fixes are wide enough". Do not let a clean pass here stand in for Δ.
