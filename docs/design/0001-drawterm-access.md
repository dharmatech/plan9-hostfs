# Design 0001: Drawterm access

- Status: Draft
- Date: 2026-07-17
- Branch: `drawterm-access`
- Depends on: Design 0002
- Related: Design 0003

## Summary

Plan 9 HostFS should provide an opt-in mode that boots a CPU server and accepts
drawterm connections from the local host. The first milestone is a standalone,
single-identity configuration that intentionally uses the compatibility
fallback in the 9front drawterm client instead of running an authentication
server.

The existing `./host/qemu` behavior remains the default. Standalone drawterm
access is selected explicitly:

```sh
./host/qemu -drawterm standalone
```

The launcher creates and reuses private NVRAM state outside the repository,
forwards only TCP 17010 on the loopback interface, and boots the existing CPU
kernel and authenticated CPU service. A future `authsrv` profile may add a real
authentication server, but it is not part of this milestone.

This document distinguishes decisions, source-supported expectations, behavior
verified by the end-to-end prototype, and checks that remain for implementation.

## Context

The current `host/qemu` launcher:

- builds `u9fs`;
- exports `root-fs/` through `u9fs -a none -u $USER`;
- boots `root-fs/boot/pxeboot.raw` with snapshot behavior;
- gives the guest access to the checkout over QEMU user networking; and
- attaches the QEMU serial console to the invoking terminal.

It does not forward a CPU or authentication port to the host. Its normal boot
path uses the terminal kernel at `root-fs/amd64/9k10`, visible to the guest as
`/amd64/9k10`.

The desired client is the 9front drawterm executable running on Windows. The
server remains Russ Cox's legacy Plan 9 environment running under QEMU in WSL.

## Goals

1. Preserve the behavior of `./host/qemu` when no drawterm option is present.
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

The repository already contains `root-fs/amd64/9k10cpuf`. Its source
configuration, `root-fs/sys/src/9k/k10/k10cpuf`, specifies `boot cpu`, includes
TCP and local boot support, sets `cpuserver = 1`, and includes the storage
drivers needed to expose an attached NVRAM disk.

The similarly named `root-fs/amd64/9k10cpu` is not suitable for this profile.
Its kernel configuration omits block-storage support, so an additional QEMU
disk cannot provide persistent NVRAM. The prototype therefore selected
`9k10cpuf`; the terminal-mode default remains `9k10`.

During CPU-server startup, `root-fs/rc/bin/cpurc` starts the TCP service
listener with `aux/listen -q tcp`. The normal CPU service at
`root-fs/rc/bin/service/tcp17010` contains:

```rc
#!/bin/cpu -R
```

This is the service the standalone profile should expose.

The special embedded root under `root-fs/sys/src/9k/k10/root` contains a different
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
initialized and boot continues. The prototype verified this flow with a 1 MiB
raw image attached as the second IDE disk and selected explicitly as:

```text
nvram=#S/sdC1/data
```

A first-use PXE configuration with `factotumopts=-k` prompted for and wrote the
credentials. A normal reuse configuration omitted `factotumopts`; the next
cold boot reused the record without prompting. No reboot between prompting and
the first CPU-service startup was required.

The inherited source tree also contains a 512-byte file named
`root-fs/sys/src/9k/root/nvram`, which may be embedded by kernel configurations
that use that root. Standalone mode does not inspect, modify, or depend on that
opaque inherited file. Its explicit `nvram` setting selects the private
external disk instead.

### Authentication-server support already present

`root-fs/rc/bin/cpurc` contains commented guidance for running `auth/keyfs` and moving
`authsrv.tcp567` into service as `tcp567`. This provides a plausible basis for
a future full authentication profile, but enabling it requires persistent key
state and a more extensive provisioning lifecycle.

### HostFS identity boundary

The launcher starts `u9fs` from `root-fs/boot` with:

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
./host/qemu

# First milestone
./host/qemu -drawterm standalone

# Reserved as a possible future interface; not implemented yet
./host/qemu -drawterm authsrv
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

Existing `-pxenew` and `-mini` behavior must not regress. Standalone mode with
`-mini` is unsupported because the embedded mini filesystem is not the HostFS
CPU-server environment; the launcher must reject that combination clearly.
Standalone mode with `-pxenew` remains an implementation check because the new
PXE image is absent from this checkout.

### Standalone authentication model

The standalone profile uses:

- the `root-fs/amd64/9k10cpuf` CPU-server kernel;
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
separate user-facing provisioning command is unnecessary: the prototype showed
that the source-supported first-boot flow is reliable and continues directly
to CPU-server startup.

Internally, the launcher needs two PXE configurations. If it created the NVRAM
image for this invocation, it boots with `factotumopts=-k` so Plan 9 owns the
interactive initialization. If the image already existed, it boots without
`factotumopts=-k`. Both configurations set `nvram=#S/sdC1/data`. Existence is
the lifecycle signal; the launcher must never replace or truncate an existing
image to force provisioning again.

### QEMU disk snapshot behavior

The current launcher uses the global QEMU `-snapshot` option. That option makes
writes temporary for disk images in the VM and would also prevent a newly
attached NVRAM disk from persisting.

QEMU 6.2 supports `snapshot=on|off` per `-drive`. The implementation should
replace the global option with equivalent per-drive behavior:

```sh
-drive file="$pxe",format=raw,snapshot=on
```

The prototype verified the following arrangement:

- the PXE boot image is IDE disk index 0 with `snapshot=on`;
- the private NVRAM image is IDE disk index 1 with `snapshot=off`;
- the NVRAM image is a 1 MiB raw file;
- the state directory is mode 0700 and the image is mode 0600; and
- `9k10cpuf` exposes the image consistently as `#S/sdC1/data`.

The implementation must express snapshot behavior per drive rather than use
QEMU's global `-snapshot`. The private image remains writable and persistent,
while the disposable PXE-disk writes retain the existing behavior.

The prototype also showed that starting the CPU service modifies the tracked
placeholder `root-fs/sys/log/listen`. Design 0003 defines the separate runtime
log boundary required for normal boots to leave the working tree clean.

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
./host/qemu -drawterm standalone
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
path. This exact command was verified with the Windows 9front drawterm build.

On first provisioning, there can be a noticeable pause after entering the
secstore password while Plan 9 writes and processes the authentication record.
The user should wait for the `gnot#` prompt before connecting drawterm.

The Windows client sends its current Windows directory to the CPU server. The
Plan 9 server cannot resolve a path such as `C:\Users\dharm`, so it briefly
prints a message like:

```text
cpu: failed to chdir to 'C:\Users\dharm'
```

It then falls back to `/usr/glenda` and starts the requested command. This is a
cosmetic behavior of the selected client/server combination, not a failed
connection, and does not require a custom drawterm build.

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

## Prototype verification results

The disposable end-to-end prototype established the following:

1. `9k10cpuf` boots as a CPU server while retaining access to the HostFS root
   and an attached block device.
2. A blank 1 MiB raw image attached as IDE disk index 1 is stable at
   `#S/sdC1/data`.
3. First boot with `factotumopts=-k` prompts for `glenda`, domain
   `plan9-hostfs`, an authentication password, and a secstore password, writes
   the record, and continues to the `gnot#` prompt without a reboot.
4. A cold restart without `factotumopts=-k` reuses the NVRAM record without
   credential prompts.
5. Only host TCP `127.0.0.1:17010` is forwarded. Neither the guest nor QEMU
   provides a listener on TCP 567.
6. The Windows 9front drawterm client connects with `-O`, reaches its local
   ticket fallback after the explicit authentication address fails, opens Rio
   as `glenda`, and starts in `/usr/glenda` on `gnot`.
7. `/README`, `/amd64`, `/cfg`, and `/sys` remain the WSL-host-backed tree in
   the drawterm session.
8. Two drawterm clients can connect simultaneously. They receive distinct
   processes, and Rio and Acme run independently in both sessions.
9. Closing both clients and QEMU, then starting a new cold QEMU process, does
   not invalidate the saved credentials.
10. The prototype's private NVRAM image can be removed without modifying any
    credential-bearing repository file.

The following checks remain for the actual launcher implementation:

1. Confirm that `./host/qemu` remains behaviorally compatible at its existing
   user interface.
2. Confirm that per-drive PXE snapshot behavior matches the original global
   `-snapshot` behavior.
3. Reject `-drawterm standalone -mini` clearly.
4. Verify `-drawterm standalone -pxenew` if a new PXE image becomes available;
   until then, fail clearly because that image is absent.
5. Implement Design 0003 so CPU-service logs do not dirty tracked files.

## Acceptance criteria

The standalone milestone is complete when all of the following hold:

- `./host/qemu` behaves as it did before the change.
- The existing image-selection options have documented, tested behavior.
- `./host/qemu -drawterm standalone` creates private state outside the
  checkout on first use.
- First use initializes NVRAM interactively without embedding a default
  password.
- Subsequent boots reuse the initialized NVRAM.
- QEMU forwards only `127.0.0.1:17010` for the standalone profile.
- No authentication port is forwarded.
- Windows 9front drawterm connects with `-O` as `glenda` and starts `rio`.
- Two simultaneous drawterm sessions can run independent Rio and Acme
  processes.
- The NVRAM file remains writable and persistent while the PXE disk retains
  snapshot behavior.
- The Windows-current-directory warning is documented as cosmetic, and the
  session falls back to `/usr/glenda`.
- Initialization and normal use leave the Git working tree clean, including
  service logs covered by Design 0003.
- Passwords and credential-bearing state do not appear in Git, logs, or process
  arguments.
- Removing disposable profile state does not require changing tracked files.

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
