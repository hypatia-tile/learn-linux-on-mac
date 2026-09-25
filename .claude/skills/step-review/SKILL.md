---
name: step-review
description: Close one roadmap step in learn-linux-on-mac — grade the owner's naming of the cause against the injected fault, or examine them on a step that carried no fault, then disclose, record and close. Done means the cause was named, never that the service runs again. Use when the owner says the diagnosis is written, asks for the review, or reports the step finished. The owner may phrase this in Japanese — 診断が書けた, レビューして, 終わった.
---

The back half of a step. `step-start` filed it, reviewed the artefacts, verified
the machine and broke it; this skill decides whether it is over.

**This skill never writes code.** It grades, discloses and records. Fixes are
the owner's.

## Required reading

1. `docs/roadmap.md` — the step's own `Done when …` clause.
2. `.faults/step-NN.md` — the ground truth. If the step carried a fault and this
   file is missing, **do not guess what was done**: say the record is gone and
   that the step cannot be graded.
3. The step's issue — the diagnosis, and how many hints were granted.
4. `CLAUDE.md` — the repository-wide rules.

If the artefacts have not been reviewed yet (`step-start`'s artefact review,
which happens before injection), that review is owed before this one. It reads
the code; this reads the diagnosis.

## Done is the naming, not the service

**A step is done when the cause has been named.** Restoring service is neither
sufficient nor, on its own, required. The next identical failure will be just as
opaque to someone who only made it go away.

A naming is correct when it has **both** halves:

1. **The change identified** — what was different, from what to what.
2. **The causal account** — how that change produced *the symptom that was
   actually observed*. Not a plausible symptom; the one in the journal.

| Owner's state | Verdict |
|---|---|
| Service restored, cause named with both halves | **done** |
| Service restored, cause not named — or named without the causal account | **failed.** Working again and not understood is the exact thing this repository exists to refuse |
| Service **not** restored, cause named with both halves | **done.** Naming it is the objective; restoring it is a separate skill |
| Service restored by compensating elsewhere, leaving the fault in place | Not a verdict but **the finding of the step.** Disclose that the original fault is still there and that there are now two deviations. This is the most common real-world accident; record it rather than punish it |

**This side declares the verdict**, because this side holds the ground truth. Say
it plainly — `done` or `failed` — and never soften a `failed` into a `done`
because service is running.

Hints do not change the verdict. They are recorded next to it: a cause named at
stage 3 is still named, and the record says so honestly.

## Steps that carried no fault

Steps 0, 1, 2, 7, 15 and 16 have no cause to name, so the rubric above does not
apply. Close them two ways at once:

**Whatever a machine can settle, settle on the machine.** Get in with
`colima ssh -p linuxlab` and look. Step 0's `/Users` genuinely absent from `ls`
and from `mount`. Step 2's `curl` answering. Step 7's and 15's service running.
The owner's report is not the evidence.

**Whatever lives only in the owner's head, examine.** Build the questions from
that step's `Done when …` clause, **on the spot** — never from a fixed question
bank, which could be read ahead. Step 1's clause asks that each directory can be
described by who writes it and what survives a reboot, so: what is the
difference between `/var/lib` and `/var/cache`; what happens to `/run` across a
reboot; why is this `/etc` a different kind of object from the Nix one on the
host. An unanswered question is a `failed` step, and the step stays open.

Steps 7 and 15 carry one more condition from the roadmap: **every gap found in
`lab.md` on the way is committed as a fix to it.** If no gap was found, put that
claim on the issue in as many words — the next rebuild will test it.

## Order of operations

The order is load-bearing. Disclosing early destroys the step.

1. **Read the diagnosis** from the issue. It must be written down before
   anything is disclosed — a diagnosis given out loud after hearing the answer
   is not evidence of anything.
2. **Grade it** against `.faults/step-NN.md`.
3. **Disclose what was actually done**, in full. Where the owner's account and
   the fault differ, **the difference is the finding** — including when the
   owner named something more interesting than what was injected.
4. **Record** (below).
5. **Close the issue.**

## Record

Everything in English, as with every artefact in this repository.

- A comment on the step's issue: the verdict, the fault as it actually was, the
  findings from the difference, and the hint stages that were reached.
- `git add .faults/step-NN.md` — it stops being a secret at disclosure, and
  becomes the permanent record of what this step's fault was.
- Tick the step's `- [ ]` in `docs/roadmap.md`. **Only on `done`.**
- Any correction to `lab.md` the step turned up.

### Committing

- **This side may run `git commit` on the owner's behalf.** The code is the
  owner's; the commit message is where the *why* is recorded, so **print every
  message in full before committing**.
- **Commits go straight to `main` and are pushed.**
- **Never amend a pushed commit.** A review is pinned to a commit hash; amending
  orphans it. Fixes to pushed work are stacked as new commits. This is not a
  style preference — it is what keeps the reviews in this repository meaningful.

## After a failed step

The issue stays open and the checkbox stays unticked. The fault has been
disclosed, so it cannot be re-run as the same exercise; what remains is the
causal account, and the step closes when the owner can give it. Do not invent a
replacement fault to salvage the grade — the next step brings one anyway.
