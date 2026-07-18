# Design 0001: Drawterm access

- Status: Draft
- Date: 2026-07-17
- Branch: `drawterm-access`

## Summary

Plan 9 HostFS should provide an opt-in mode that boots a CPU server and accepts
drawterm connections from the local host. The first milestone is a standalone,
single-identity configuration that intentionally uses the compatibility
fallback in the 9front drawterm client instead of running an authentication
server.

The existing `./boot/qemu` behavior remains the default. Standalone drawterm
access is selected explicitly:

```sh
./boot/qemu -drawterm standalone
```

The launcher creates and reuses private NVRAM state outside the repository,
forwards only TCP 17010 on the loopback interface, and boots the existing CPU
kernel and authenticated CPU service. A future `authsrv` profile may add a real
authentication server, but it is not part of this milestone.

This document distinguishes decisions from source-supported expectations and
items that still require an end-to-end prototype.

## Context

The current `boot/qemu` launcher:

- builds `u9fs`;
- exports the repository root through `u9fs -a none -u $USER`;
- boots `boot/pxeboot.raw` with snapshot behavior;
- gives the guest access to the checkout over QEMU user networking; and
- attaches the QEMU serial console to the invoking terminal.

It does not forward a CPU or authentication port to the host. Its normal boot
path uses the terminal kernel at `amd64/9k10`.

The desired client is the 9front drawterm executable running on Windows. The
server remains Russ Cox's legacy Plan 9 environment running under QEMU in WSL.

## Goals

1. Preserve the behavior of `./boot/qemu` when no drawterm option is present.
2. Add an explicit standalone drawterm mode using the legacy CPU protocol.
3. Bind host forwarding to `127.0.0.1` only.
4. Forward only the port required by the selected profile.
5. Persist credentials across boots without tracking credential state in Git.
6. Let the user choose the password interactively in Plan 9.
7. Keep the launcher compatible with POSIX `/bin/sh`.
8. Document the trust and identity limitations of the standalone profile.

## Non-goals

The first milestone does not provide:

- a Plan 9 authentication server;
- general multi-user authentication or authorization;
- distinct Unix ownership for different Plan 9 identities;
- a remotely reachable CPU service;
- the embedded `cpu -N` no-authentication service;
- 9front guest support;
- non-amd64 guest support; or
- a Windows-native or NTFS-backed HostFS implementation.

## Investigation findings

### CPU-server boot path

The repository already contains `amd64/9k10cpu`. Its source configuration,
`sys/src/9k/k10/k10cpu`, specifies `boot cpu`, includes TCP boot support, and
sets `cpuserver = 1`.

During CPU-server startup, `rc/bin/cpurc` starts the TCP service listener with
`aux/listen -q tcp`. The normal CPU service at
`rc/bin/service/tcp17010` contains:

```rc
#!/bin/cpu -R
```

This is the service the standalone profile should expose.

The special embedded root under `sys/src/9k/k10/root` contains a different
`tcp17010` service:

```rc
#!/bin/cpu -N -R
```

That service belongs to the separate `k10root` configuration and is not the
normal HostFS QEMU boot path. The standalone design does not switch to it.

### Drawterm protocol selection and fallback

The local 9front drawterm client tries the newer `rcpu` protocol on TCP 17019
before falling back to the legacy CPU path. Its `-O` option skips `rcpu` and
selects the legacy CPU protocol on TCP 17010. Legacy Plan 9 HostFS connections
must therefore use `-O` explicitly.

For legacy CPU authentication, this drawterm first attempts to obtain tickets
from an authentication server. If that attempt fails, `gettickets` calls
`mkservertickets`. That compatibility path succeeds only when the relevant
server and host identities match. The standalone profile is designed around a
single `glenda` identity and must be verified end to end with that client.

This fallback is a feature of the selected 9front drawterm implementation. The
standalone profile is not a promise that every historical or third-party
drawterm client can connect without an authentication server.

### NVRAM initialization

CPU boot starts factotum in server mode. In that mode, factotum requests NVRAM
with `NVwriteonerr`: a missing or invalid NVRAM record can trigger interactive
credential prompts and a write of the new record.

The legacy `readnvram` implementation accepts an explicit `nvram` device. For
an explicit raw device that is not a `9fat` partition, it uses offset zero. Its
interactive data includes an authentication identity, authentication domain,
authentication password, and secstore password.

The source therefore supports a first-run flow in which a blank disk is
initialized and boot continues. Whether the selected QEMU disk attachment and
this kernel complete that flow without a reboot remains a prototype test, not
a confirmed behavior.

### Authentication-server support already present

`rc/bin/cpurc` contains commented guidance for running `auth/keyfs` and moving
`authsrv.tcp567` into service as `tcp567`. This provides a plausible basis for
a future full authentication profile, but enabling it requires persistent key
state and a more extensive provisioning lifecycle.

### HostFS identity boundary

The current launcher starts `u9fs` with:

```sh
u9fs -a none -u "$USER" ..
```

All attaches therefore use the invoking Unix identity. Even a future auth
server could authenticate multiple Plan 9 users without automatically giving
them different Unix ownership or permission enforcement in the exported host
tree. Future work should be called `authsrv`, not `multi-user`, until this
separate HostFS identity problem is designed.

## Decisions

### One repository with runtime profiles

Runtime behavior belongs behind launcher options in one repository. Separate
forks and long-lived branches are development mechanisms, not user-facing
configuration.

The intended interface is:

```sh
# Existing terminal behavior
./boot/qemu

# First milestone
./boot/qemu -drawterm standalone

# Reserved as a possible future interface; not implemented yet
./boot/qemu -drawterm authsrv
```

The unimplemented `authsrv` value must not be accepted silently. Unknown or
unavailable profiles should produce a clear usage error.

The launcher is the control point because it must select the kernel, QEMU
disks, and host port forwarding before the guest boots. A guest-only setting
such as `plan9.ini` cannot control all three concerns by itself.

### Preserve the default path

With no `-drawterm` option, the launcher must continue to use the existing
terminal kernel, network export, PXE disk, serial console, and lack of host
forwarding.

Existing `-pxenew` and `-mini` behavior must not regress. Compatibility between
those image-selection options and standalone mode will be made explicit during
implementation; unsupported combinations must fail clearly rather than select
an accidental boot path.

### Standalone authentication model

The standalone profile uses:

- the `amd64/9k10cpu` CPU-server kernel;
- the normal `cpu -R` service on guest TCP 17010;
- `authid` `glenda`;
- `authdom` `plan9-hostfs`;
- a password chosen by the user during first boot; and
- the intentional 9front drawterm ticket fallback.

It does not start `keyfs` or `authsrv`, and it does not forward guest TCP 567.

### Persistent state location

Credential-bearing state must be outside the checkout. The default state root
is:

```text
${XDG_STATE_HOME:-$HOME/.local/state}/plan9-hostfs
```

`PLAN9_HOSTFS_STATE` overrides that root. Profile state is isolated beneath it:

```text
plan9-hostfs/
    standalone/
        nvram.img
```

Future profiles receive separate subdirectories. State directories should be
mode 0700 and credential-bearing files mode 0600, subject to the host
filesystem's capabilities.

The launcher must never place a password in command-line arguments, diagnostic
output, or the repository. It must not silently reset existing NVRAM. Any
future reset command must identify the exact target and require confirmation.

### First-run lifecycle

On the first standalone boot, the launcher creates a blank NVRAM disk if one
does not already exist. Plan 9 then owns credential initialization through its
normal console prompts. Subsequent standalone boots reuse that disk.

The expected initial values are:

```text
authid: glenda
authdom: plan9-hostfs
auth password: chosen by the user
secstore password: chosen by the user as appropriate
```

No initialized NVRAM image or shared default password is distributed. A
separate provisioning command will be added only if the prototype shows that
the source-supported first-boot flow is unreliable or requires a reboot.

### QEMU disk snapshot behavior

The current launcher uses the global QEMU `-snapshot` option. That option makes
writes temporary for disk images in the VM and would also prevent a newly
attached NVRAM disk from persisting.

QEMU 6.2 supports `snapshot=on|off` per `-drive`. The implementation should
replace the global option with equivalent per-drive behavior:

```sh
-drive file="$pxe",format=raw,snapshot=on
```

The standalone NVRAM drive must be writable with snapshot behavior disabled.
Its exact controller, QEMU device declaration, image size, and stable Plan 9
device name remain prototype decisions.

### Minimal loopback forwarding

Standalone mode forwards only host TCP 17010 to guest TCP 17010 and binds the
host listener to loopback:

```sh
cpu_fwd='hostfwd=tcp:127.0.0.1:17010-:17010'
```

QEMU accepts repeated `hostfwd` suboptions inside one user-mode `-netdev`.
For the initial single forward, the intended form is:

```sh
-netdev "user,id=net0,$u9fs,$tftp,$cpu_fwd"
```

If later profiles need several forwards, each forward should retain a named
variable for documentation and those values may be assembled into one
`hostfwds` variable before constructing `-netdev`.

Do not add speculative forwards. In particular, standalone mode does not
forward 567, 564, 17019, or any of the auxiliary ports used by unrelated
9front experiments.

## Planned user workflow

### First boot

```sh
./boot/qemu -drawterm standalone
```

The user completes the Plan 9 NVRAM prompts at the QEMU serial console and
keeps QEMU running.

### Windows drawterm connection

The planned explicit client form is:

```powershell
$env:PASS = '<the password chosen during first boot>'
C:\Users\dharm\src\drawterm\build\msvc\drawterm.exe `
    -O `
    -h 'tcp!127.0.0.1!17010' `
    -a 'tcp!127.0.0.1!567' `
    -u glenda `
    -c rio
```

`-O` is mandatory because this guest provides legacy CPU rather than 9front
`rcpu`. The explicit authentication address documents where the client's first
ticket attempt goes; standalone mode deliberately has no listener or QEMU
forward on that port, so the selected client must take its local compatibility
path. The exact Windows quoting and observed fallback failure mode will be
confirmed during acceptance testing.

## Security properties and limitations

1. The CPU port is reachable only from the host loopback interface.
2. The service still uses legacy Plan 9 CPU authentication; it is not `cpu -N`.
3. Standalone mode is intentionally single-identity and depends on client
   fallback behavior.
4. Anyone with local access to the NVRAM state or its password should be
   treated as able to authenticate as `glenda`.
5. The NVRAM file is a secret-bearing mutable artifact and should be protected
   and backed up accordingly.
6. The profile is not suitable for exposure on a LAN or public network.
7. Plan 9 identities do not form distinct Unix security principals through the
   current `u9fs -a none -u "$USER"` export.

## Prototype verification items

The following are intentionally unresolved in this draft:

1. Determine the cleanest way to select `amd64/9k10cpu` without changing the
   default PXE boot behavior.
2. Choose a QEMU disk/controller arrangement that gives the NVRAM disk a stable
   Plan 9 device name.
3. Choose the smallest robust NVRAM image size and set the correct `nvram`
   value for the guest.
4. Confirm that first boot prompts, writes NVRAM, and continues without a
   reboot.
5. Confirm that a second boot reuses NVRAM without prompting.
6. Confirm that the Windows 9front drawterm client reaches the intended
   fallback and opens `rio` as `glenda`.
7. Confirm that the QEMU host listener is bound only to `127.0.0.1:17010`.
8. Confirm that replacing global `-snapshot` with per-drive `snapshot=on`
   preserves the existing PXE-disk behavior.
9. Define and test valid combinations with `-pxenew` and `-mini`.

## Acceptance criteria

The standalone milestone is complete when all of the following hold:

- `./boot/qemu` behaves as it did before the change.
- The existing image-selection options have documented, tested behavior.
- `./boot/qemu -drawterm standalone` creates private state outside the
  checkout on first use.
- First use initializes NVRAM interactively without embedding a default
  password.
- Subsequent boots reuse the initialized NVRAM.
- QEMU forwards only `127.0.0.1:17010` for the standalone profile.
- No authentication port is forwarded.
- Windows 9front drawterm connects with `-O` as `glenda` and starts `rio`.
- The NVRAM file remains writable and persistent while the PXE disk retains
  snapshot behavior.
- Initialization and normal use leave the Git working tree clean.
- Passwords and credential-bearing state do not appear in Git, logs, or process
  arguments.

## Alternatives considered

### Commit a preinitialized NVRAM image

Rejected. It would distribute a shared credential, track mutable binary state,
and make accidental credential commits likely.

### Use `cpu -N`

Rejected for this profile. The no-auth service belongs to a special embedded
root configuration, bypasses the intended authentication path, and is not what
the selected drawterm client naturally targets.

### Implement a full authentication server first

Deferred. `keyfs`, `/adm/keys`, `authsrv`, user provisioning, and recovery add
significant state and lifecycle complexity. They should follow a working
standalone baseline.

### Use separate forks or branches for runtime modes

Rejected. Users should choose a profile at launch rather than change source
history to choose behavior.

### Store state inside the repository

Rejected. The repository is exported into the guest, and credential state must
not be visible as ordinary project content or appear in Git status.

### Forward a broad set of Plan 9 ports

Rejected. Ports will be added only for an implemented, documented service.

### Keep QEMU's global `-snapshot`

Rejected for standalone mode because it would make NVRAM writes temporary.
Per-drive snapshot policy preserves the immutable PXE behavior and permits a
persistent credential disk.

## Future work

### `authsrv` profile

A future design may add:

- profile-specific key and NVRAM state;
- `keyfs` and `authsrv` startup;
- loopback forwarding from host TCP 17567 to guest TCP 567;
- an explicit drawterm authentication address such as
  `tcp!127.0.0.1!17567`;
- user provisioning and credential rotation; and
- backup and recovery procedures.

That work must separately address the fact that current `u9fs` attaches all
map to one Unix user.

### Other hosts and guests

Later designs may investigate 9front, arm64 guests, macOS acceleration, and a
Windows-native control or filesystem backend. Those projects should reuse the
profile and state concepts where appropriate without expanding the scope of
the standalone milestone.
