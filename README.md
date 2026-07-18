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
[`docs/design`](docs/design/README.md). Drawterm access is designed there but
is not yet implemented.

## Lineage and licensing

The original bootable 9legacy/QEMU integration comes from Russ Cox's fork.
Plan 9 HostFS is an independent downstream project; its changes and decisions
should not be attributed to Russ.

The project-root [`LICENSE`](LICENSE) and [`NOTICE`](NOTICE) are exact copies
of the primary Plan 9 files under `root-fs/`. Additional licenses and notices
required by the guest distribution remain at the root of `root-fs/`.
