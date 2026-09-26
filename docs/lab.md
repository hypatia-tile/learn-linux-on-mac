# The lab

`linuxlab` is a colima profile kept separate from the default one, so that
breaking it does not take the working Docker setup with it.

This file is the **only** record of how the VM is built. Colima keeps its
instance state and its generated `colima.yaml` under `~/.colima`, which is
outside Nix's management — nothing here is reproduced by
`darwin-rebuild switch`. If a procedure is not written down here, it is lost at
the next `colima delete`. That is what Steps 7 and 15 test.

The colima commands in this file are the one thing the AI supplies — the single
narrow exception in `.claude/skills/step-start`. Everything *inside* the VM is
the owner's.

## What is already on the host

Verified 2026-09-26:

| | |
|---|---|
| `colima` | 0.10.3, from Nix at `/etc/profiles/per-user/kazukishinohara/bin/colima` |
| that path | a **bash wrapper**, not the binary. The real one is `.colima-wrapped` beside it, and the wrapper is what puts `lima-full`, `qemu`, `docker` and `krunkit` on colima's own `PATH` |
| `docker` | 29.8.0, same place. **Unused by this roadmap** — the VM runs with no container runtime until Step 16 |
| `limactl` | **not on `PATH`** — see the wrapper above |
| colima state | **`~/.config/colima/`**. `~/.colima` does not exist and never did |

Colima's state layout, since every path below is outside Nix's management:

| path | what it is |
|---|---|
| `~/.config/colima/<profile>/colima.yaml` | the generated colima config for that profile — what was *asked for* |
| `~/.config/colima/_lima/colima-<profile>/lima.yaml` | the lima config colima produced from it — what was *built*. The default profile's instance is named `colima`, not `colima-default` |
| `~/.config/colima/_lima/_networks/` | lima's network configuration |
| `~/.config/colima/ssh_config` | the generated ssh entries |

`~/.ssh/config` is not edited on every start. It carries a single
`Include ~/.config/colima/ssh_config` line, added when a profile was first
started; colima then maintains the included file. The generated entry for a
profile is `Host colima-<profile>`, pointing at `127.0.0.1` on a
per-instance port.

## Creating it

```sh
colima start linuxlab \
  --runtime none \
  --mount none \
  --cpu 2 --memory 4 --disk 60 \
  --network-address
```

Flag by flag, because each one is load-bearing:

- **`--mount none`** — no host directory is shared into the VM. Colima's default
  is the opposite, and an empty `mounts: []` in `colima.yaml` is *not* the same
  as no mount: it means "use the default", and the default is stated in colima's
  own generated config as *"Colima default behaviour: `$HOME` is mounted as
  writable."* Since the AI has `sudo` in here, leaving that would give it write
  access to the entire host home directory.
- **`--runtime none`** — no container runtime. The VM is a machine, not a
  container host, until Step 16 installs a runtime by hand. **This value is
  fixed at creation** — `colima.yaml` says so in as many words — so it is the
  one flag that cannot be corrected by a restart.
- **`--cpu` / `--memory` / `--disk`** — colima's defaults are 2 CPU / 2 GiB /
  100 GiB. 4 GiB leaves room for nginx alongside the daemon; 60 GiB is more than
  enough for the loop-device volumes of Step 10. All three are settable on a
  later start.
- **`--network-address`** — assign the VM an address reachable from the host.
  Needed from Step 6 onward. Also settable later, and removable with
  `--network-address=false`.

`--activate=false` is **not** in the list, although it looks like it should be.
Under `runtime: none` there is nothing to activate — no Docker context, no
Kubernetes context, no Incus remote — and no socket is created for the profile.
`autoActivate: true` stays in the generated config and is inert. Passing the flag
would document a fear rather than a fact.

### As built

The live `linuxlab` was not created with the full command above: `--runtime none
--mount none` first, then `--memory 4 --network-address` on a later start. It
therefore stands at **disk 100 GiB** (colima's default) rather than 60. Nothing
is wrong with it — the roadmap needs *at least* 60 — and only the runtime would
have required a rebuild.

### Flags persist

`--save-config` defaults to true, so every flag passed to `colima start` is
written into that profile's `colima.yaml` and applies to later starts.
Consequences worth knowing before they surprise someone:

- A flag is removed by passing its opposite (`--network-address=false`), not by
  omitting it.
- `colima start <profile> --edit` opens the config in `$EDITOR` before starting,
  which is the honest way to see what a flag actually set.

## What Step 0 settled

Three of the five open questions this file carried were answered from the host,
without entering the VM. The two that remain are the ones that matter, and they
are answered from inside.

**1. `--runtime none` is accepted.** The flag's own help enumerates
`(containerd, docker, incus)` and the binary carries `unsupported container
runtime '%s'`, which together read like a whitelist and are not one: the
enumeration lists the runtimes colima can *manage*, while `none` is the absence
of one and is handled outside that registry. `colima list` reports
`RUNTIME none` for the profile. Of the two comments inside the generated
`colima.yaml`, the one calling `none` a runtime was accurate; the one
enumerating `(docker, containerd)` is incomplete.

**2. The mount is absent — from the host's side.** The generated lima config is
the evidence, and the two profiles make the contrast plain:

| profile | `_lima/<instance>/lima.yaml` |
|---|---|
| `default`, started with no flags | `mounts:` → `- location: "~"`, `writable: true`, `mountType: virtiofs` |
| `linuxlab`, `--mount none` | **no `mounts:` key at all** |

This is worth having as a positive control, but it is not the proof. The proof
is from inside the VM, and it is the owner's to produce — **still open**.

**3. `--network-address` costs nothing on this host.** It assigns an address
(`192.168.64.5` on the live instance) using lima's **user-v2** network, because
the VM type is `vz`. No `socket_vmnet`, no entry under `/etc/sudoers.d/`, no
password prompt. The `error setting up sudoers for route` string in the binary
belongs to the qemu path, not this one.

**4. Nothing colima-owned is activated.** The host's docker context stays at
`default`, and `~/.config/colima/linuxlab/` holds only `colima.yaml` — no
`docker.sock`. This is a property of `runtime: none`, not of a flag.

**5. What the guest is** — `/etc/os-release`, the kernel, and that PID 1 is
systemd. **Still open**, and Step 1 starts from the answer.

### One discrepancy, left standing on purpose

`colima list` reports `linuxlab`'s disk as **100 GiB**, the value in
`colima.yaml`. The instance's `lima.yaml` says **`disk: 20GiB`**. One is the
configuration and one is the instance; this file does not yet know which governs
the block device the guest actually sees. **Re-check before Step 10 sizes its
loop devices**, from inside the guest rather than from either config.

## Using it

```sh
colima ssh -p linuxlab          # get in
colima status -p linuxlab       # what it is doing
colima restart -p linuxlab      # bounce it
colima stop -p linuxlab         # leave it for later
colima delete -p linuxlab       # destroy it — expected at Steps 7 and 15
```

Omitting `-p` does not mean "the profile I was last using" — it means the
`default` profile, which is the working Docker setup this lab was kept away
from. `colima start` with no arguments starts *that*.

`ssh -F ~/.config/colima/ssh_config colima-linuxlab` reaches the same VM without
going through colima, which matters once the daemon needs to be reached from the
host.

## Getting code into the VM

There is no shared directory, by design. The repository arrives over the
network:

```sh
git clone https://github.com/hypatia-tile/learn-linux-on-mac.git
```

This is how software reaches a real server, so the inconvenience points in the
right direction.
