# Design 0002: Repository layout

- Status: Accepted
- Date: 2026-07-17
- Branch: `repository-layout`
- Depends on: none
- Affects: Design 0001

## Summary

Plan 9 HostFS will separate project metadata and host tooling from the Plan 9
filesystem exported to the guest. The current Git repository and its complete
history will be retained, but the Plan 9 tree will move beneath `root-fs/` and
the host launcher will move to `host/qemu`.

The target layout is:

```text
plan9-hostfs/
    .gitignore
    docs/
        design/
    host/
        qemu
    root-fs/
        386/
        amd64/
        boot/
        cfg/
        rc/
        sys/
        ...
        README
        LICENSE
        LICENSE.*
        NOTICE
        NOTICE.old
    README.md
    LICENSE
    NOTICE
```

`root-fs/` becomes the single filesystem root used by both `u9fs` and QEMU
TFTP. Project documentation, Git metadata, and host-only tools are not visible
inside Plan 9.

This project does not treat future merges from `rsc/plan9` as a design goal.
The upstream history remains valuable for provenance and file-level history,
so the migration will use ordinary Git moves rather than rewriting or
discarding history.

## Context

Russ Cox's repository deliberately uses one directory for two roles:

1. the Git project root; and
2. the Plan 9 root filesystem exported to QEMU.

Its README describes booting with the Git clone as the root filesystem. The
current `boot/qemu` launcher changes into `boot/` and uses `..` for all of the
following:

- the `u9fs` export root;
- the QEMU TFTP root; and
- paths to host-built `u9fs` and boot images.

That arrangement is compact, but it mixes project concerns with guest
filesystem contents. New project files such as `docs/`, host scripts, and Git
metadata become visible to Plan 9 even though they are not part of the guest
environment.

Plan 9 HostFS is becoming an independently organized downstream project. Its
layout should make the boundary between the host project and the exported
guest filesystem explicit.

## Findings

### `u9fs` has one export root

The bundled `u9fs` accepts an optional filesystem root argument and serves that
tree. It does not provide an exclusion option for hiding selected project-root
entries. A clean export boundary therefore requires a distinct directory or a
custom filesystem-view mechanism.

The distinct-directory approach is simpler, easier to inspect, and does not
require a permanent fork of `u9fs` solely for project-file filtering.

### QEMU uses the same tree for TFTP

The current QEMU user-network configuration sets `tftp=..`. PXE loads the
configuration at `cfg/pxe/525400123456`, and that configuration names kernels
with guest-root paths such as `/amd64/9k10`.

After restructuring, QEMU's host-side TFTP root must be `root-fs/`. The
contents and guest-visible paths under that root remain unchanged.

### The launcher is host tooling

`boot/qemu` is a POSIX host shell script, not a Plan 9 command. The raw PXE
images in `boot/` are guest boot assets, but the launcher itself belongs on the
host side of the project boundary.

The launcher will therefore move to `host/qemu`, while `boot/mini.raw` and
`boot/pxeboot.raw` move with the guest tree to `root-fs/boot/`.

### Licensing files are guest-visible inputs

The current root contains:

- `LICENSE`;
- `LICENSE.afpl`;
- `LICENSE.gpl`;
- `LICENSE.lucida`;
- `LICENSE.old`;
- `NOTICE`; and
- `NOTICE.old`.

These are not merely GitHub metadata. `sys/lib/dist/mini/proto` includes the
license and notice files in the mini distribution, and bundled documentation
refers to absolute guest paths such as `/LICENSE.gpl`.

All existing license and notice files must therefore remain available at the
Plan 9 filesystem root, which means moving them into `root-fs/` without content
changes. Exact copies of the primary `LICENSE` and `NOTICE` may also remain at
the project root for repository tooling and immediate attribution. No legal
text will be rewritten as part of the layout migration.

### Windows cannot perform the migration faithfully

The tree contains hundreds of paths with components named `aux`, which is a
reserved Windows device name, and it relies extensively on executable mode
bits. All filesystem and Git move operations must be performed inside WSL on
the Linux filesystem. The migration must not enumerate or move this tree
through a Windows checkout or UNC filesystem operation.

### Verified pre-migration baseline

The inherited layout was booted before any tracked file was moved. Both launch
modes were run through the unmodified `boot/qemu` script under WSL QEMU 6.2.0.
Menu option 2 (1440x900) was used consistently.

#### Default PXE and HostFS boot

The default launcher:

- built the bundled `u9fs` successfully;
- loaded `cfg/pxe/525400123456` through QEMU TFTP;
- loaded `amd64/9k10` after selecting bootfile option 2;
- reached Rio as user `glenda` on sysname `gnot`;
- started in `/usr/glenda`; and
- opened a live host `u9fs -a none -u dharmatech ..` process.

Guest inspection confirmed that `/README.md`, `/docs/design`, and the complete
`/.git` directory were visible through the exported repository root. This is
the behavior the new `root-fs/` boundary removes. The guest's `/boot` was
overlaid by the boot filesystem at runtime, hiding the host launcher's current
`boot/` directory after startup.

The boot changed no tracked files. It produced only ignored u9fs objects and
binary output, `sys/log/dns`, and files beneath `usr/glenda/tmp`.

#### Mini boot

The `-mini` launcher:

- booted `boot/mini.raw` through its disk loader;
- loaded the embedded compressed 386 `9pc` kernel;
- spent several minutes in `kfs...gunzip...` under software emulation but
  continued successfully;
- reached Rio as `glenda` on `gnot`, starting in `/usr/glenda`; and
- did not open a host u9fs process.

The mini root is therefore an embedded filesystem snapshot, not a HostFS
export. Its old timestamps and `rsc`/`staff` ownership distinguish it from the
live checkout. Its source manifest at `sys/lib/dist/mini/proto` explicitly
includes the primary README and legal files.

#### New PXE image

`boot/pxeboot.raw.new` is absent from this checkout. The `-pxenew` option could
not be boot-tested before migration and is expected to continue selecting the
corresponding relocated path. Until an image is supplied, both layouts should
fail clearly at QEMU image open rather than silently selecting another image.

#### Stopping the guest

This configuration is a net-booted terminal with a remote u9fs root, not a
standalone machine with a writable local file server. Legacy `fshalt` syncs
and halts local Fossil, Venti, or KFS services and does not power off this
guest. The kernel's `reboot` command resets the virtual machine instead of
closing QEMU.

The normal HostFS stop procedure is therefore to finish active writes, wait
for the command prompt, and close the QEMU window. This procedure belongs in
the project README. Closing QEMU during an active edit, copy, or build can
still interrupt an individual host-file write and must not be described as
transactional shutdown.

## Goals

1. Give the exported Plan 9 filesystem one explicit host directory.
2. Keep project documentation, `.git`, and host-only tooling outside the
   exported tree.
3. Retain the full Git commit history and GitHub fork lineage.
4. Preserve guest-visible paths such as `/sys`, `/amd64`, and `/cfg`.
5. Preserve the behavior of the default QEMU boot after its command moves.
6. Make host paths independent of the caller's working directory.
7. Keep every existing license and notice available at its expected guest
   path.
8. Make the migration reviewable and recoverable as an ordinary commit.

## Non-goals

This design does not attempt to:

- support future automatic merges from `rsc/plan9`;
- rewrite, squash, or discard existing Git history;
- split the project into multiple repositories or submodules;
- change Plan 9 guest paths;
- change the contents or legal meaning of license and notice files;
- rebuild or replace the existing binary `mini.raw` image;
- add drawterm behavior;
- add other guest operating systems; or
- provide a general-purpose filesystem overlay or export filter.

## Decision

### Project identity

Plan 9 HostFS is an independently organized downstream distribution that
retains `rsc/plan9` ancestry. Future upstream merging is not a layout
constraint.

The project remains in the current GitHub fork and retains all parent commits.
The migration is represented by normal commits on top of the existing history.

### Directory names

The exported tree is named `root-fs/`. The hyphen makes the host-side concept
explicit while the directory name itself is never visible inside the guest.

Host-only launch and management tools live under `host/`. The initial launcher
path is:

```text
host/qemu
```

The launcher keeps no filename extension and remains executable with
`#!/bin/sh`.

### Target layout

The intended top-level structure is:

```text
plan9-hostfs/
    .git/
    .gitignore
    docs/
        design/
            README.md
            0001-drawterm-access.md
            0002-repository-layout.md
    host/
        qemu
    root-fs/
        386/
        9load
        acme/
        adm/
        amd64/
        arm/
        boot/
        cfg/
        cron/
        dist/
        env/
        fd/
        lib/
        lp/
        mail/
        mips/
        mips64/
        mnt/
        n/
        pbsraw
        power/
        power64/
        rc/
        riscv/
        riscv64/
        sparc/
        spim/
        spim64/
        sys/
        tmp/
        usr/
        LICENSE
        LICENSE.afpl
        LICENSE.gpl
        LICENSE.lucida
        LICENSE.old
        NOTICE
        NOTICE.old
        README
    README.md
    LICENSE
    NOTICE
```

The abbreviated tree shows the current top-level guest entries. The migration
inventory must be generated from the tracked tree immediately before moving so
that no tracked guest entry is accidentally omitted.

### Move and stay inventory

The following stay outside `root-fs/`:

- `.git/`;
- `.gitignore`;
- `docs/`;
- `README.md`, rewritten as the Plan 9 HostFS project README;
- `host/`; and
- exact project-root copies of the primary `LICENSE` and `NOTICE`.

The existing plain `README` moves to `root-fs/README`.

All current Plan 9 filesystem directories move beneath `root-fs/`, including
architecture directories, `cfg`, `rc`, `sys`, `tmp`, and `usr`.

The executable guest-root files `9load` and `pbsraw` also move beneath
`root-fs/` without content or mode changes.

The existing `boot/qemu` moves to `host/qemu`. The remainder of `boot/` moves
to `root-fs/boot/`.

All existing `LICENSE*` and `NOTICE*` files move to `root-fs/` so guest and
distribution references continue to resolve. The project-root `LICENSE` and
`NOTICE` are exact copies, not replacements or edited summaries.

The mini filesystem manifest currently embeds both `README` and `README.md`.
The plain `README` is Plan 9 distribution documentation and remains embedded.
The Markdown file is host-facing QEMU project documentation and will be
removed from the manifest. The already-shipped `boot/mini.raw` remains
byte-for-byte unchanged during this migration; rebuilding binary boot media is
a separate, explicit operation.

### Host path contract

`host/qemu` must determine the project root from its own path rather than the
caller's current directory. Conceptually:

```text
project root = parent of the launcher directory
root fs      = <project root>/root-fs
```

All host-side paths derive from those locations:

- build directory: `root-fs/sys/src/cmd/unix/u9fs`;
- `u9fs` export root: `root-fs`;
- TFTP root: `root-fs`;
- default PXE image: `root-fs/boot/pxeboot.raw`;
- new PXE image: `root-fs/boot/pxeboot.raw.new`; and
- mini image: `root-fs/boot/mini.raw`.

Paths must be quoted correctly for POSIX shell use. The QEMU `guestfwd` command
contains an embedded host command and requires specific verification when the
checkout path contains spaces or shell-significant characters.

The documented default command becomes:

```sh
./host/qemu
```

There is no compatibility wrapper at `boot/qemu`, because `boot/` belongs
inside the exported tree and retaining host tooling there would recreate the
boundary this design removes.

### Guest path contract

Moving the host directory does not add a `/root-fs` directory inside Plan 9.
`u9fs` serves the contents of the host's `root-fs/` as guest `/`.

The following guest paths remain unchanged:

```text
/386
/amd64
/boot
/cfg
/rc
/sys
/usr
/LICENSE
/LICENSE.gpl
/NOTICE
```

Project-only paths must not be present in the normal guest root:

```text
/.git
/docs
/host
/README.md
```

### Ignore rules

General object and archive patterns in `.gitignore` continue to match beneath
`root-fs/`. Explicit paths must be updated for the new host location, including:

```text
root-fs/usr/glenda/tmp
root-fs/sys/log
root-fs/sys/src/cmd/unix/u9fs/u9fs
```

Generated host state, including NVRAM, remains outside the repository as
specified by Design 0001 and must not be added to `.gitignore` as a substitute
for correct state placement.

### Git history and migration commits

The migration will not use an orphan branch, `filter-repo`, history rewriting,
or a new repository.

The current pre-layout commit may receive an annotated tag, tentatively
`rsc-plan9-snapshot`, after separate approval. The tag is a convenience; the
parent commit already preserves the complete original layout.

The intended commit sequence is:

1. Record the approved design documents.
2. In one coherent migration commit, perform the Git moves and the minimum
   launcher, ignore, README, and path changes required to leave the default
   boot operational.
3. Update Design 0001 for the accepted paths before drawterm implementation.

Keeping the migration operational in one commit avoids inserting a known
broken commit between a pure move and its path fixes. Most files remain
byte-for-byte identical, allowing Git's rename detection and `git log --follow`
to preserve useful file history.

No migration commit or tag is created merely by accepting this design. Each
Git mutation remains a separately approved action.

### Branch sequence

Repository layout is a prerequisite for drawterm implementation and deserves a
branch whose name matches that scope. The intended sequence is:

1. Rename or replace the current branch with `repository-layout`.
2. Approve and implement this design there.
3. Integrate the layout into the project's main line.
4. Create a fresh `drawterm-access` branch from the restructured tree.
5. Update and then implement Design 0001.

The exact branch operations will be proposed and approved separately. This
document does not rename a branch.

## Migration procedure

The implementation should follow these safeguards:

1. Confirm the current branch, commit, and full working-tree status.
2. Confirm the exact tracked top-level inventory.
3. Ensure design-document changes are preserved before moving tracked files.
4. Perform all operations inside WSL on the Linux filesystem.
5. Create `root-fs/` and `host/` only at the validated repository path.
6. Move `boot/qemu` to `host/qemu` with Git.
7. Move the explicit, reviewed guest inventory beneath `root-fs/` with Git.
8. Preserve every executable mode and file byte.
9. Create exact top-level copies of the primary `LICENSE` and `NOTICE`.
10. Update `.gitignore`, `host/qemu`, and `README.md` for the new paths.
11. Remove host-facing `README.md` from `root-fs/sys/lib/dist/mini/proto`
    without rebuilding or modifying the existing `mini.raw` image.
12. Review Git's rename detection and investigate unexpected additions or
    deletions.
13. Run the acceptance checks before staging or committing.

Do not use a broad wildcard or a Windows-side recursive move. The target
directory and every top-level source entry must be resolved before the move so
that `.git`, `docs`, and newly created project directories cannot be moved by
accident.

## Acceptance criteria

The repository-layout milestone is complete when all of the following hold:

- The top level contains only the approved project and host entries.
- Every previously tracked guest filesystem entry is present under
  `root-fs/`, except `boot/qemu`, which is at `host/qemu`.
- All upstream license and notice files exist at their expected guest-root
  paths.
- The project-root `LICENSE` and `NOTICE` exactly match their primary
  `root-fs/` counterparts.
- `./host/qemu` can be invoked from the repository root.
- `host/qemu` also works when invoked through an absolute path from another
  working directory.
- The default VM boots with the same terminal behavior as before.
- `-mini` reaches the same embedded Rio environment as before.
- `-pxenew` selects `root-fs/boot/pxeboot.raw.new` and, while that image is
  absent, fails clearly rather than falling back to another image.
- The existing `root-fs/boot/mini.raw` is byte-for-byte unchanged.
- QEMU TFTP loads configuration and kernels from `root-fs/`.
- `u9fs` exports `root-fs/` as guest `/`.
- Guest `/sys`, `/amd64`, `/cfg`, and `/LICENSE.gpl` resolve normally.
- Guest `/.git`, `/docs`, `/host`, and `/README.md` are absent.
- Building `u9fs` leaves only ignored generated output.
- A sample `git log --follow` crosses the layout commit for a moved source
  file.
- The mini manifest retains `README` and legal files but no longer embeds the
  host-facing project `README.md`.
- Git reports no unexpected content modifications among mechanically moved
  guest files.
- Design 0001 is updated before drawterm implementation begins.

## Rollback

The migration occurs on a dedicated branch and does not rewrite history. Until
it is integrated, abandoning that branch leaves the original layout unchanged.

After integration, the migration should remain revertible as one coherent
commit. The pre-layout parent commit, and an optional annotated tag, provide an
unambiguous reference for inspection and recovery.

Rollback must use ordinary Git history operations chosen for the situation; it
must not rely on destructive filesystem deletion or history rewriting.

## Alternatives considered

### Keep the Git root as the guest root

Rejected for this project direction. It preserves upstream organization but
continues to expose project metadata and constrains the placement of host
tools, documentation, and future support files.

### Add exclusions to `u9fs`

Rejected as the primary layout mechanism. It would keep the working tree
intermixed, require maintaining filtering behavior, and leave QEMU TFTP with a
separate view problem.

### Create a generated or mounted export tree

Rejected for now. Bind mounts, overlay filesystems, generated mirrors, or
synchronization add privileges, state, and failure modes while weakening the
immediate host/guest editing model.

### Create an outer repository with a submodule

Rejected. It gives a clean boundary and easy upstream tracking but introduces
two repositories, recursive clone requirements, and cross-repository change
coordination. Upstream merging is not important enough to justify that cost.

### Discard or rewrite Git history

Rejected. History does not affect current tree cleanliness and remains useful
for provenance, blame, auditing, and understanding the imported source.

### Retain `boot/qemu` as a compatibility wrapper

Rejected. A host-only file under guest `/boot` contradicts the chosen boundary.
This project has not released the new layout, so a clean command transition is
preferable to permanent compatibility clutter.

## Effect on Design 0001

Design 0001 was drafted against the inherited layout. After this design is
accepted and implemented, Design 0001 must be revised as follows:

- replace `./boot/qemu` with `./host/qemu`;
- replace host references such as `amd64/9k10cpu` with
  `root-fs/amd64/9k10cpu`;
- replace host references such as `rc/bin/cpurc` with
  `root-fs/rc/bin/cpurc`;
- keep guest paths unchanged;
- describe `root-fs/` as the `u9fs` and TFTP root; and
- preserve the rule that NVRAM and credentials live outside the repository.

Design 0001 remains a draft and must not be implemented against paths known to
be temporary.

## Remaining verification items

1. Confirm the full tracked move inventory immediately before migration.
2. Verify shell and QEMU quoting when the project path contains spaces.
3. Confirm Git rename detection across the large directory move.
4. Re-run the default and mini boot baselines after migration.
5. Treat any pre-layout annotated tag as a separate future decision; no tag is
   part of this milestone.
