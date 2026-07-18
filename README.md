# Plan 9 HostFS

Plan 9 HostFS is an experimental, bootable distribution of legacy Plan 9 that
uses a normal host directory as the guest's root filesystem. It runs under
QEMU on Linux, including Linux in WSL.

The project is derived from
[Russ Cox's `rsc/plan9`](https://github.com/rsc/plan9), which combined a
bootable 9legacy tree with QEMU, PXE/TFTP, and `u9fs`. This
fork keeps that unusually direct host/guest editing model while separating
project files and host tools from the filesystem exported to Plan 9.

The Plan 9 distribution's own introduction is in
[`root-fs/README`](root-fs/README).

## Requirements

The current launcher expects a Linux environment with:

- `qemu-system-x86_64`;
- `make`; and
- a C compiler for building the bundled `u9fs`.

On Windows, run the project from a Linux filesystem in WSL. The Plan 9 tree
contains names and executable modes that cannot be represented faithfully by
a normal Windows checkout.

## Run

```sh
git clone https://github.com/dharmatech/plan9-hostfs.git
cd plan9-hostfs
./host/qemu
```

The launcher may also be invoked by absolute path or from another working
directory. It resolves `root-fs/` relative to itself.

The default boot loads `root-fs/cfg/pxe/525400123456` over QEMU TFTP and then
loads the selected kernel from `root-fs/`. The amd64 bootfile is
`ether0!/amd64/9k10`.

At the current startup menus:

1. choose the display resolution you want; and
2. choose `ether0!/amd64/9k10` for the normal 64-bit HostFS guest.

The launcher builds `root-fs/sys/src/cmd/unix/u9fs/u9fs` automatically.
Generated build and runtime files remain ignored by Git.

## Drawterm access

Standalone drawterm mode boots the legacy CPU service with one authenticated
Plan 9 identity:

```sh
./host/qemu -drawterm standalone
```

On the first run, choose menu option 1 and enter:

```text
authid: glenda
authdom: plan9-hostfs
auth password: a private password of your choice
secstore password: the same private password
```

The final password values are not displayed. After entering the secstore
password, initialization can pause for a while. Wait for the `gnot#` prompt
before starting drawterm.

The launcher stores the private NVRAM image outside the checkout at:

```text
${XDG_STATE_HOME:-$HOME/.local/state}/plan9-hostfs/standalone/nvram.img
```

Set `PLAN9_HOSTFS_STATE` to an absolute directory to override the
`plan9-hostfs` state root. State directories are protected with mode 0700 and
the 1 MiB NVRAM image with mode 0600. Subsequent boots reuse the initialized
image without prompting for credentials.

Keep QEMU running and connect from Windows PowerShell with the 9front drawterm
client:

```powershell
$drawterm = 'C:\path\to\drawterm.exe'
$env:PASS = '<the auth password chosen during first boot>'
& $drawterm `
    -O `
    -h 'tcp!127.0.0.1!17010' `
    -a 'tcp!127.0.0.1!567' `
    -u glenda `
    -c rio
```

`-O` selects the legacy CPU protocol. The explicit authentication address has
no listener in standalone mode; the selected 9front client falls back to its
local single-identity ticket path. QEMU forwards only host loopback TCP 17010.
It does not forward an authentication port or expose the CPU service on the
LAN.

The Windows client sends its current directory to the CPU server. Plan 9 may
briefly report a message such as:

```text
cpu: failed to chdir to 'C:\Users\you'
```

This is cosmetic. The session falls back to `/usr/glenda` and starts Rio.
Multiple drawterm clients can connect to the same QEMU instance and run
independent Rio and Acme sessions.

Standalone mode is intentionally a single-identity local development profile,
not a multi-user authentication server. Anyone with the password or NVRAM
state should be treated as able to authenticate as `glenda`. Do not expose the
CPU port beyond loopback.

Only one standalone QEMU process may use a state directory at a time. The
launcher records this with `standalone/qemu.lock` and removes the lock on a
normal exit. If an abnormal termination leaves that directory behind, first
confirm that no standalone QEMU process is running. Remove only
`qemu.lock/pid`, then remove the empty `qemu.lock` directory named in the
launcher's error message.

If first-run provisioning is interrupted before NVRAM initialization
completes, stop QEMU before changing state. Removing `standalone/nvram.img`
discards the standalone credentials and makes the next launch provision a new
image. Back up an initialized NVRAM image if losing those credentials would be
inconvenient.

Standalone mode is incompatible with `-mini`. The optional `-pxenew` image may
be combined with it when that image is available; the current checkout does
not include `pxeboot.raw.new`.

## Host filesystem model

QEMU exports `root-fs/` through `u9fs`, and Plan 9 mounts its contents as `/`.
For example:

```text
host                              Plan 9 guest
root-fs/sys/src                   /sys/src
root-fs/usr/glenda                /usr/glenda
root-fs/cfg/pxe/525400123456      /cfg/pxe/525400123456
```

Completed host and guest edits are visible on both sides immediately. Project
metadata and host tooling stay outside the export, so `/.git`, `/docs`,
`/host`, and the project `README.md` are not part of the normal guest root.

All Plan 9 file access through the current `u9fs -a none` export uses the Unix
identity that launched QEMU. Plan 9 user names do not create distinct Unix
security principals on the host.

## Mini environment

```sh
./host/qemu -mini
```

Mini mode boots the self-contained 386 environment stored in
`root-fs/boot/mini.raw`. It does not mount the live HostFS tree. Decompressing
its embedded filesystem can take several minutes when QEMU is using software
emulation.

The existing `mini.raw` is retained as an upstream binary artifact. Its source
manifest embeds the Plan 9 distribution README and legal files, but not this
host-facing project README.

The `-pxenew` option selects `root-fs/boot/pxeboot.raw.new`. That optional
image is not included in the current checkout, so the option will fail clearly
unless the image is supplied.

## Stopping QEMU

Finish active edits, copies, and builds, wait for the Plan 9 command prompt,
and then close the QEMU window.

This guest is a net-booted terminal with a remote `u9fs` root. Legacy
`fshalt` is for syncing and halting local Plan 9 file servers and does not
power off this configuration. The `reboot` command resets the virtual machine
instead of closing QEMU.

Closing QEMU while a host-backed file is actively being written can still
leave that individual operation incomplete.

## Design and development

Architectural decisions and verified baselines are recorded in
[`docs/design`](docs/design/README.md). Standalone drawterm access and the
HostFS runtime-state boundary are implemented from those designs.

## Lineage and licensing

The original bootable 9legacy/QEMU integration comes from Russ Cox's fork.
Plan 9 HostFS is an independent downstream project; its changes and decisions
should not be attributed to Russ.

The project-root [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE) are exact copies
of the primary Plan 9 files under `root-fs/`. Additional licenses and notices
required by the guest distribution remain at the root of `root-fs/`.
