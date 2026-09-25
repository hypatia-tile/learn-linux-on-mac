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

Verified 2026-09-25:

| | |
|---|---|
| `colima` | 0.10.3, from Nix at `/etc/profiles/per-user/kazukishinohara/bin/colima` |
| `docker` | 29.8.0, same place |
| `limactl` | **not on `PATH`** — the nixpkgs colima wrapper puts `lima-full`, `qemu`, `docker` and `krunkit` on colima's own `PATH` only |
| `~/.colima` | did not exist; colima had never been started |

## Creating it

```sh
colima start linuxlab \
  --runtime none \
  --mount none \
  --activate=false \
  --network-address \
  --cpu 2 --memory 4 --disk 60
```

Flag by flag, because each one is load-bearing:

- **`--mount none`** — no host directory is shared into the VM. Colima's
  default is the opposite, and its own embedded config says so: *"Colima
  default behaviour: `$HOME` is mounted as writable."* Since the AI has `sudo`
  in here, leaving that default would give it write access to the entire host
  home directory.
- **`--runtime none`** — no container runtime. The VM is a machine, not a
  container host, until Step 16 installs a runtime by hand.
- **`--activate=false`** — do not make this profile the active Docker or
  Kubernetes context. The default profile keeps that job.
- **`--network-address`** — assign the VM an address reachable from the host.
  Needed from Step 6 onward, when the service has to be reached from outside.
- **`--cpu` / `--memory` / `--disk`** — colima's defaults are 2 CPU / 2 GiB /
  100 GiB. 4 GiB leaves room for nginx alongside the daemon; 60 GiB is more
  than enough for the loop-device volumes of Step 10.

## Verify at Step 0

These are written from colima's flags and embedded documentation, not from a
run. Step 0 is where they become facts — correct this file where they are
wrong, and commit the correction.

1. **Is `--runtime none` accepted?** Colima's config comment documents only
   `(docker, containerd)`, while its `rootDisk` comment refers to *"the `none`
   runtime"*. The two disagree. If the flag is rejected, fall back to
   `--runtime containerd` and disable the service inside the VM, and record
   that here.
2. **Is the mount genuinely absent?** Do not take it on trust. From inside:
   `ls /Users`, and `mount` — the host home directory must not appear.
3. **What does `--network-address` do here?** It may ask for the host password
   on first use. Record the address it assigns.
4. **Did `--activate=false` hold?** `docker context ls` on the host must still
   point at the default profile.
5. **What is the guest?** Record `/etc/os-release` — the roadmap assumes
   `apt`, and Step 1 starts from what is actually there.

## Using it

```sh
colima ssh -p linuxlab          # get in
colima status -p linuxlab       # what it is doing
colima stop -p linuxlab         # leave it for later
colima delete -p linuxlab       # destroy it — expected at Steps 7 and 15
```

Colima starts with `sshConfig: true`, so it edits `~/.ssh/config` on start.
Worth knowing before it surprises someone.

## Getting code into the VM

There is no shared directory, by design. The repository arrives over the
network:

```sh
git clone https://github.com/hypatia-tile/learn-linux-on-mac.git
```

This is how software reaches a real server, so the inconvenience points in the
right direction.
