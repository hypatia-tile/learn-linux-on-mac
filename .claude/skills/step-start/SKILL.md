---
name: step-start
description: Run the front half of one roadmap step in learn-linux-on-mac — file the step issue and present the spec, review what the owner built, verify it on the machine, then inject the fault. State-dispatched, so the same skill covers every mid-step moment. Use when the owner asks to start, resume or move on to the next step, says the work is ready, asks for it to be reviewed or broken, or asks for a hint. The owner may phrase any of this in Japanese — 次の step, 始めて, できた, 見て, 壊して, 仕込んで, ヒント.
---

The front half of a step: from filing it to handing the machine back broken.
The back half — grading the diagnosis, disclosing the fault, closing the step —
is `step-review`.

A step spans days and many sessions, so this skill has no single entry point.
**Dispatch on state**, not on the words used to invoke it.

## Required reading, every invocation

1. `docs/roadmap.md` — the goal and the steps. Nothing else; the operating
   rules live here now.
2. `docs/lab.md` — how `linuxlab` is built and reached. The colima commands in
   it are the one thing this side supplies.
3. `CLAUDE.md` — the repository-wide rules (language, host read-only, git).
4. [`design.md`](https://github.com/hypatia-tile/plan-learn-linux-on-mac/blob/main/docs/design.md)
   in `plan-learn-linux-on-mac`, when the *why* of a step is unclear. The
   reasoning is there, not in the roadmap.

## Who writes what

- **The owner writes everything that runs on the machine.** The C daemon, every
  unit file, the nginx configuration, every shell script. Naming a file or a
  setting is specification; writing its body is not. This is the whole point of
  the repository — the code is not the deliverable, the ability to write it is.
- **This side writes `docs/roadmap.md`, the step issues, the reviews** — and the
  colima procedure in `lab.md`. That last one is the single exception, and it is
  deliberately narrow: **colima commands only**. It does not extend to systemd,
  nginx, C, or anything inside the VM.
- **This side injects the faults.** It may do anything inside `linuxlab`,
  including `sudo` and including destroying it. The host stays read-only —
  `CLAUDE.md` holds that rule.

## Dispatch

Establish state first: `gh issue list --state open`, `git status --short`, and
whether `.faults/step-NN.md` exists for the open step.

| State | What this skill does |
|---|---|
| No step issue open | **File and spec** — the first unchecked step in the roadmap |
| Issue open, owner is still building | Nothing. Say so and stop |
| Owner says the work is ready | **Review the artefacts**, then **verify on the machine**, then **inject** |
| Fault injected, owner asks for a hint | **Apply the clock** |
| Fault injected, owner has a diagnosis | Hand over to `step-review` |

**Only one step issue is open at a time.** Concurrent steps would cross the
`.faults/` records and the hint clocks. If a second step is asked for while one
is open, refuse and push for the open one to be closed.

## File and spec

Target the first unchecked (`- [ ]`) step in `docs/roadmap.md`. One issue per
step; the issue is where the diagnosis will be written.

The spec **contains**:

- The file paths and the settings the step turns on, by name — `User=`,
  `Restart=`, `WorkingDirectory=`, which directory the mutable state goes in.
- What has to be decided, and what the decision has to be defended against.
  Step 3 is not "write a unit file", it is "defend where the binary, the
  configuration and the state each belong".
- The step's own done condition, copied from the roadmap's `Done when …`.
- Which commands will demonstrate it, so the owner knows what evidence to keep.

The spec **never contains**:

- **The body of anything that runs on the machine.** Not a unit file, not a C
  function, not an nginx block, not a shell one-liner that does the work. Naming
  `Restart=` is specification; writing `Restart=on-failure` is doing the step.
- Any hint about the fault that is coming. The fault is chosen later — after the
  artefact review — precisely so that it cannot leak into the spec.

Look every fact up before writing it. The guest distribution, what a flag
actually accepts, what is on the host: read it or run it. `lab.md` records
several claims that were written from documentation and are marked for
correction at Step 0 — do not compound that.

## Review the artefacts

When the owner says the work is ready, read what was actually written, pinned to
the current `HEAD`. This happens **before** the fault, not after, and it is the
most valuable half hour of the step.

The question is never "does it work" — it does, or the owner would not have said
it was ready. The question is **"can this choice be defended?"**:

- `Restart=always` against `on-failure`, and what each does to a service that
  fails at startup rather than in flight.
- `Type=simple` against `notify`, and what systemd believes in each case.
- Where the state directory is, and whether `StateDirectory=` would have made
  the `mkdir` unnecessary.
- In C: signal handling, a `write()` whose return value is dropped, an `errno`
  swallowed, a buffer sized by assumption.

Ask. Do not fix. A choice the owner cannot defend is a finding, and it is
recorded on the issue.

**Findings here feed the fault.** A weakness that surfaced in this review is the
best possible fault for this step — it is aimed at something known to be soft,
which is worth more than any amount of blindness. Prefer it over a fault picked
from a list.

## Verify on the machine

Never inject on the strength of the owner's word. Get in and see it:

```sh
colima ssh -p linuxlab
```

Confirm the step's done condition is actually met — the unit is enabled and
running, the journal carries the lines, `curl` answers. If it is not met, stop
here and say what is missing. Breaking something that was already broken
produces a step that cannot be graded.

## Inject

One fault per step. Announce it at the level the step's phase allows, and
nothing beyond:

| Phase | Steps | Disclosed at injection time |
|---|---|---|
| 1 | 3–4 | The exact surface — "one line in the unit file was changed" |
| 2 | 5–8 | The area only — "either the user or the permissions" |
| 3 | 9–16 | Nothing |

### Traces are left alone

**Do not cover tracks.** mtime stays, root's shell history stays, `sudo` in the
authentication log stays. Finding the file that changed three minutes ago and
reading who became root are legitimate operator skills, and suppressing them
would teach a problem that exists only because there is an AI in the loop.

**The journal is never altered.** Not truncated, not rewritten, nothing
fabricated into it. journald is the instrument being learned; an instrument that
lies teaches nothing.

The defence against a step ending at one `ls -lt` is not concealment, it is
**variety**. Keep faults that touch no file content in rotation:

- ownership or a permission bit, leaving mtime of the contents untouched
- a running process killed, or runtime state broken with the configuration intact
- a filesystem filled (Step 11's subject)
- an nftables rule or a route dropped from the running kernel only — **the
  configuration on disk is clean, so a reboot fixes it**. "It came back after a
  restart and nobody knows why" is the worst real-world ending there is, and it
  is worth walking into once, on purpose, in Phase 3.

### The record

Write `.faults/step-NN.md` at injection time. **Leave it untracked** — it shows
up in `git status`, which is the point: it is in plain sight and the owner does
not read it. `step-review` commits it once the diagnosis is in.

```markdown
# Step NN — fault

- **Injected**: 2026-09-26T14:32:00+09:00
- **Phase / disclosure**: 3 — nothing disclosed
- **Changed**: /etc/systemd/system/httpd.service, `User=svc` → `User=svcd`
- **Before**: (the exact previous value, verbatim)
- **Expected symptom**: unit fails to start, 217/USER, nothing in the daemon's log
- **The naming that counts as correct**: the unit runs as a user that does not
  exist, so systemd fails before `exec`, which is why the daemon's own log is
  empty rather than showing an error
- **How to undo**: restore `User=svc`, `systemctl daemon-reload`, restart
```

The injection timestamp is what the hint clock is computed from, so it is not
optional.

**Every fault must be undoable by the written procedure.** The one exception is
Step 12, where destruction is the subject and the backup is the only way
back — and there, the backup's existence is verified before anything is
destroyed.

## Apply the clock

Hints are clock-driven, and **never offered**. If the owner is visibly stuck and
has not asked, stay quiet — that silence is the exercise.

When asked, compute the elapsed time from the `Injected` timestamp in
`.faults/step-NN.md` and apply the stage mechanically:

| Elapsed | Hint |
|---|---|
| under 30 min | None. Say so |
| 30 min | Where to look. Nothing about what is there |
| 60 min | The area, narrowed |
| after that | The answer, and the step continues from there |

Before granting one, ask for **one line on what has been learned since the last
checkpoint**. "Stuck" means no new information, not unsolved. Having read the
journal is progress: hold the hint and say which line of that thread to pull.

The owner may skip the clock. If they ask for the answer outright, give it —
that is theirs to spend, not a failure. Record it as the step having reached
stage 3.

Every hint granted goes on the issue as a comment, with its stage and the time.
`step-review` weighs the diagnosis against how many hints it took.

## What this skill does not do

It does not declare a step done, grade a diagnosis, disclose what the fault was,
tick a roadmap checkbox, or commit. All of that is `step-review`.
