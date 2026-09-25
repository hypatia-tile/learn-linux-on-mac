# CLAUDE.md

Operator-level Linux, learned on a disposable colima VM by growing one service
and repeatedly being handed it broken. The owner writes everything that runs on
the machine; this side specifies, reviews, and breaks it.

## Running a step

The whole operating procedure lives in two skills, and they are the authority —
not this file, and not `docs/roadmap.md`, which carries only the goal and the
steps.

- **`.claude/skills/step-start`** — file the step, spec it, review what was
  built, verify it on the machine, inject the fault, apply the hint clock.
- **`.claude/skills/step-review`** — grade the naming of the cause, disclose the
  fault, record, close.

**Signpost.** A step spans days and many sessions, so most invocations land
mid-step with no skill loaded. If the owner says any of 「できた」「見て」
「壊して」「仕込んで」「ヒント」— or their English equivalents — **read
`step-start` before answering**. The rules for those moments are in it, and
answering without them will break the exercise.

`.faults/step-NN.md` appearing as untracked in `git status` is the record of the
open step's fault. **Do not read it** when acting for the owner; it is left in
plain sight on trust, and `step-review` commits it once the diagnosis is in.

## Language

**Everything in this repository is written in English** — `README.md`, the docs,
comments in `.c` files, unit file comments, issue bodies, review comments and
commit messages. This holds even when the conversation is in Japanese.

## The host is read-only

Inside `linuxlab`, anything goes — `sudo`, destruction, whatever a fault needs
(`step-start` sets the terms). **On the host, read-only commands only**, exactly
as in every other repository.

The VM is started with `--mount none`, so the host home directory is not
reachable from inside. Colima's default is the opposite, so this is not left to
chance; `docs/lab.md` has the flags and why each one is load-bearing.

## Where the reasoning is

The *why* behind the plan — a VM and not a container, faults injected by someone
else, a hand-written daemon — is in
[`plan-learn-linux-on-mac`](https://github.com/hypatia-tile/plan-learn-linux-on-mac).
Read its `docs/design.md` before questioning the shape of a step.
