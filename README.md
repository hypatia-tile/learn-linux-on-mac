# learn-linux-on-mac

Learning operator-level Linux from macOS, on a disposable colima VM, by growing
one service and repeatedly being handed it broken.

- **[`docs/roadmap.md`](docs/roadmap.md)** — the goal and the steps.
- **[`CLAUDE.md`](CLAUDE.md)** — the repository-wide rules, and the two skills
  under `.claude/skills` that run a step from start to close.
- **[`docs/lab.md`](docs/lab.md)** — how `linuxlab` is built, and what has to
  be verified about it.
- **`src/`** — the small HTTP daemon in C that the whole roadmap is hung on.
- **`etc/`** — the unit files and configuration that get deployed into the VM.

The reasoning behind the plan — why a VM and not a container, why the faults
are injected by someone else, why the daemon is hand-written rather than
installed — is in
**[`plan-learn-linux-on-mac`](https://github.com/hypatia-tile/plan-learn-linux-on-mac)**.

## The rule that matters

A step is done when the cause has been **named**, not when the service runs
again. Restoring service without being able to say what was wrong is a failed
step.
