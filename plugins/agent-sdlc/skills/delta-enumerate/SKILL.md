---
name: delta-enumerate
description: Gate Δ of the agent-sdlc chain — enumerate every line this round's gates changed, because the lines you just fixed are the one place sampling structurally cannot see. Fix sub-threshold findings in place, then run one bounded confirmation pass (Δ′) over only those fix lines; escalate anything that reaches a dangerous path or overturns a written guarantee to a second gate 7. Use after external-ai-review (or after any review round that adopted at least one finding), before committing.
---

# Δ — enumerate every line this round changed

Sampling reviewers — external AI, human review, random probing — **structurally cannot find
"the fix you just wrote is not wide enough"**, because that spot looks already-handled and
nobody samples it twice. It is also the **highest-yield** spot in the diff: whoever wrote that
fix had, moments earlier, been shown to be thinking too narrowly about that exact axis.

This gate exists to enumerate that population. It is not another review round.

## When to use

- Right after gate 10 `/agent-sdlc:external-ai-review`, before gate 8 `/agent-sdlc:commit-message`.
- After any review round that **adopted at least one finding**.
- The user says "did you check your own fixes", "Δ", "enumerate what changed this round".

**Skip only when gates 9/10 adopted _zero_ findings** — then the population is empty. Write
"Δ empty, skipped" in the progress file.

> ⛔ **There is no "if needed".** Adopting even one finding makes this gate mandatory.
> Measured 2026-08-09: the ⏭ line got hand-written as "Δ (if needed)" from memory, the round
> had adopted 9 findings, and Δ was nearly skipped. Check whether you are adding a softener
> ("if needed" / "as appropriate" / "optional") that the rule does not contain.

## 1. Define the population mechanically — do not recall it

The population is **every line gates 5/6/7/9/10 changed _this round_**, not the whole diff.
Derive it, don't remember it:

```bash
# The clean tree as of the last commit or the pre-review snapshot, vs the working tree now.
git diff <the-diff-you-saved-before-gate-5> HEAD
# or, when the round is uncommitted work on top of a base:
git diff main -- <files touched this round>
```

If a diff of the pre-review state was saved to scratch (recommended: save one before gate 5),
restore a clean tree and diff against it. Write the resulting **file count and ± line count**
into the progress file — that number is what makes the run replayable.

Then split it: **executable changes** vs **comments/docs/controls**. Both are in scope. Prose
that *defines an observation window, a coverage claim, or a guarantee* is executable-equivalent
— it is what the next person will trust instead of re-measuring.

## 2. Enumerate sibling values — do not sample again

For each decision point inside the population, enumerate the **siblings**: adjacent spellings,
boundary values, the other members of the same axis, the encodings of the same byte. Run them
against the real code path; do not reason about them.

Measured examples of what this catches and sampling did not:

- A guard bound to two literal strings → the trailing-slash sibling still forwarded a real
  Bearer token. **The two controls written for that fix only exercised those two spellings, so
  both printed PASS.**
- A guard changed from literal match to *prefix* match — still a byte comparison, when the axis
  was *segment identity*. 3 more holes (`//auth/...`, `auth./...`, `auth../...`), all carrying
  credentials.
- The round after that, the whole byte axis was enumerated instead: all 67 allowed bytes
  appended to the guarded prefix, driven through the real decision chain → 0 forwards. **That
  is the difference between "I tested 8 spellings" and "the axis is closed".**

⚠️ **Enumerating "the axis" only helps if you drew the axis correctly.** If the same shape keeps
recurring after enumeration, the axis is wrong, not the sample size.

## 3. Threshold — by nature, never by count

**Escalate** when ≥1 finding **actually reaches a dangerous path, or overturns a guarantee
already written down**. That triggers a full second gate 7 (`/code-review max -fix`), and
**Δ is not allowed to fix those itself** — that class of defect needs a full review, not a
patch from the same head that just wrote the bug.

Nits, style, and readability **never** escalate, however many there are.

> Count is a bad metric here: one hole that carries credentials off-box proves this round's
> judgement was too narrow; three nits stacked into an escalation just burns a gate 7.

## 4. Sub-threshold findings: fix in place, then Δ′ (max 2 passes)

Fix them now. Then run **Δ′**: the same enumeration, over a population consisting **only of the
lines Δ itself just changed**. Cap at **2 passes total**.

> 🔴 **Why fixing is allowed, and why this still terminates.** The rule used to be
> "report only, never auto-fix — *this is what makes the recursion terminate*". That bound
> *termination* to *not fixing*, but they are separable: the explosion comes from
> "fix → re-enumerate **everything** → fix" having no upper bound, not from fixing. Δ′'s
> population is only the lines Δ touched and it runs once — a constant bound.
>
> The old rule's measured cost (2026-08-09): all three of Δ's findings were sub-threshold, so
> per the rule they went into the commit **unchanged** — sentences already known to be wrong
> entered version control to preserve a discipline that was not needed. And the first time Δ′
> ran under the new rule it **immediately caught a fourth**, of the same class, written while
> fixing the third. Nothing else in the chain would ever have seen it.

⚠️ **Fagan's Follow-up is preserved, not discarded**: the progress file must record
**what Δ fixed** and **what Δ left for human review** as two separate lists. Gate 11 verifies
those two lists, not an impression.

## 5. Write it down

The progress file must carry:

- **population**: which lines (file count, ± lines, how it was derived)
- **what was enumerated**: the axes and their sibling sets
- **findings**: how many, and each one's class
- **threshold**: met or not, and the reasoning
- **fixed by Δ** / **left for human review** — two separate lists
- **Δ′ result**, and that it closed within the 2-pass cap

⛔ **"Δ ran and found nothing" and "Δ did not run" must not look the same in the file.**

## Anti-patterns

- **Re-reading the diff for "anything wrong"** — that is another sampling round. Δ is
  enumeration over a *named* population.
- **Enumerating only the executable lines.** Every measured recurrence of the classic failure
  (4 times in one 2026-08-09 round) was a *prose* claim about an observation window or a
  coverage boundary, written while fixing something else.
- **Treating a clean external-review round as evidence the fixes are wide enough.** It means
  the reviewer found nothing new *by sampling*. Measured: a round that ended 0 blocker / 0 warn
  was immediately followed by Δ finding 3 and Δ′ finding a 4th.
- **Fixing an above-threshold finding here.** Escalate it.

## After this gate

<!-- DELIBERATE DELTA vs upstream appleboy/skills: sdlc callback is CONDITIONAL on a progress file existing (standalone single-skill use isn't forced into the full lifecycle). Intentional — don't restore the unconditional "REQUIRED" form. -->

**Running the full agent-sdlc lifecycle?** If a `docs/sdlc/<feature>-<pr>-sdlc-progress.md` exists (or you deliberately started the whole SOP chain), invoke `/agent-sdlc:sdlc` — it ticks this gate and reports the exact ⏭ next step. It navigates only; it will not run the next gate.

> ⛔ If the threshold was met, the next step is **a second gate 7** (`/code-review max -fix`),
> then back here — **not** gate 8. Say which of the two you are handing off, explicitly.

**Used this skill standalone?** You're done — do NOT invoke `/agent-sdlc:sdlc` (there is no progress file for it to update). If you want to keep going, the step that normally follows is **gate 8 — `/agent-sdlc:commit-message`**, then gate 11 (🚦 human review & merge).
