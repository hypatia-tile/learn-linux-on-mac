# Roadmap

Learning operator-level Linux on a disposable colima VM, by growing one service
and repeatedly being handed it broken.

Read [`design.md`](https://github.com/hypatia-tile/plan-learn-linux-on-mac/blob/main/docs/design.md)
in `plan-learn-linux-on-mac` first. The steps implement it; the reasoning is
there, not here. The lab itself is [`lab.md`](lab.md).

## How a step is run

This file holds the goal and the steps, and nothing about the procedure. Who
writes what, how the faults are injected, how blind each phase is, when a hint
is owed, and what makes a step done are all in the repository's own skills:

- **`.claude/skills/step-start`** — filing the step, the spec, the artefact
  review, verification on the machine, injection, the hint clock.
- **`.claude/skills/step-review`** — grading the named cause, disclosure,
  recording, closing.

A rule written in two places is a rule that will eventually disagree with
itself, so it is written there and not here. The one line worth repeating:

> **A step is done when the cause has been named, not when the service runs.**

## Steps

### Phase 1 — learning the instruments

- [ ] **0. Stand up the lab.** `colima start linuxlab --runtime none
      --mount none`, reach it over ssh, and **confirm the host home directory
      is genuinely not visible from inside**. Write the first version of
      `lab.md`. No fault. Done when the VM is reachable and the absence of the
      mount has been demonstrated rather than assumed.

- [ ] **1. Walk the machine.** `/etc/os-release`; PID 1 is systemd;
      `systemctl list-units`; reading `journalctl` — time ranges, `-u`, `-p`,
      `-b`. Then walk the hierarchy on the real thing: `/etc`, `/var/log`,
      `/var/lib`, `/var/cache`, `/run`, `/tmp`, `/usr/bin` against
      `/usr/local/bin`. Compare this `/etc` with the Nix one on the host — a
      symlink farm and a directory administrators edit are not the same object.
      No fault. Done when each directory can be described by who writes it and
      what survives a reboot.

- [ ] **2. Run the daemon by hand.** Install a toolchain with `apt`, clone the
      repository into the VM, build, run it in the foreground, reach it with
      `curl` from inside. No unit file yet, no fault. Done when it answers a
      request and the terminal it was started from still owns it.

- [ ] **3. Write the unit file.** Decide where the binary, the configuration
      and the daemon's mutable state each belong, and defend each choice.
      Write the unit — `User=`, `Restart=`, `WorkingDirectory=` — and
      `systemctl enable --now` it.
      *Fault (surface disclosed): one line in the unit file is changed.*
      Done when `systemctl status` and `journalctl -u` alone produce the cause.

- [ ] **4. Put the log where logs go.** stdout and stderr into journald,
      priorities, `journalctl -u -f`, and a journal that survives a reboot.
      *Fault (surface disclosed): a single change stops the log arriving.*
      Done when the silence itself has been traced to its cause.

### Phase 2 — the area, and nothing more

- [ ] **5. Give the service its own user.** A system user with `nologin`,
      ownership of the state directory, and a binary the service user cannot
      rewrite.
      *Fault (area: user or permissions).*

- [ ] **6. Ports and reachability.** `ss`, bind addresses, reaching the service
      from the host, and what it takes to listen below 1024 without running as
      root.
      *Fault (area: network reachability).*

- [ ] **7. Rebuild, first checkpoint.** `colima delete -p linuxlab`, then
      restore the Step 6 state from `lab.md` and the repository alone. No
      fault. Done when the service is running again — and every gap found in
      `lab.md` on the way is committed as a fix to it.

- [ ] **8. Close the machine down.** nftables, default-deny, and only what the
      service needs.
      *Fault (area: firewall or port).*

### Phase 3 — nothing disclosed

- [ ] **9. Ordering and dependencies.** `After=`, `Requires=`, `Wants=`, and
      what `systemd-analyze critical-chain` shows.
      *Fault: it works when started by hand and fails after a reboot.*

- [ ] **10. Give the data its own volume.** A file-backed block device with
      `dd` and `losetup`, LVM on top, a filesystem, and the state directory
      mounted there through `/etc/fstab`.
      *Fault: blind.*

- [ ] **11. Fill the disk.** `logrotate`, journald's own size limits, and what
      a full filesystem does to a service that was not expecting one.
      *Fault: blind — and it will not look like a disk problem.*

- [ ] **12. Back it up and lose it.** A backup of the state directory on a
      systemd timer, then a restore drill.
      *Fault: blind, and destructive. The backup is the only way back.*

- [ ] **13. Put nginx in front.** A reverse proxy, two units that depend on
      each other, and a self-signed certificate.
      *Fault: blind — a 502, which is a symptom belonging to either side.*

- [ ] **14. Link against something.** Grow a library dependency, then deploy
      it: `ldd`, `/usr/local/lib`, `ldconfig`, rpath.
      *Fault: blind — the binary that built fine will not start.*

- [ ] **15. Rebuild, second checkpoint.** From nothing to the Step 14 state,
      using only `lab.md` and the repository. No fault.

- [ ] **16. Take a container apart.** Install a container runtime inside
      `linuxlab` by hand. Start a container, then look at it from the machine's
      side: `/proc/<pid>/ns/*`, the cgroup tree, what `ps` on the host VM sees.
      Then build the smallest possible one by hand with `unshare`. No fault.
      Done when the namespaces a container is made of can be listed from
      memory, and the hand-made one runs.
