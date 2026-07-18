# Design 0003: HostFS runtime state

- Status: Accepted
- Date: 2026-07-17
- Branch: `drawterm-access`
- Depends on: Design 0002
- Affects: Design 0001 and future runtime profiles

## Summary

Plan 9 HostFS exports `root-fs/` directly from a Git checkout. That live model
is the project's defining feature, but it also lets guest services modify host
files that currently serve both as versioned filesystem inputs and mutable
runtime state.

The drawterm prototype exposed the first concrete collision: starting the CPU
service appended records to the tracked, initially empty file
`root-fs/sys/log/listen`. A successful boot therefore left the source tree
dirty even though no source or configuration had changed.

Mutable service logs will become generated, ignored files beneath
`root-fs/sys/log/`. The host launcher will create the required empty log files
from a versioned manifest before starting QEMU, without truncating existing
logs. Credential-bearing profile state remains outside the checkout as defined
by Design 0001.

## Context

Design 0002 made `root-fs/` the explicit tree exported by `u9fs` and used by
QEMU TFTP. Ordinary Plan 9 edits intentionally update that host tree
immediately. This is desirable for source, documentation, and user files, but
not every writable guest path is project content.

Legacy Plan 9 ships placeholder files under `/sys/log` so services can open
their named logs. The imported repository tracks 32 named files there. Of
those, 29 are zero-length placeholders and three (`mail`, `runq`, and `smtp`)
contain historical runtime records. Once a service writes a new record, Git
sees a source-file modification.

This problem is not unique to drawterm. DNS, connection servers,
authentication services, mail services, and later profiles may all write
runtime records. Solving it in the drawterm launcher alone would obscure the
general HostFS boundary.

## Findings

### Plan 9 `syslog` does not create missing files

`root-fs/sys/src/libc/9sys/syslog.c` opens `/sys/log/<name>` for writing. It
does not create a file when that path is absent. Depending on the caller, a
missing file causes the message to be written to the console or discarded.

The empty files are therefore functional inputs, not meaningless historical
artifacts. Simply removing them from Git and leaving them absent would change
guest behavior.

### The imported inventory includes historical log data

The three nonempty tracked files contain mail-service activity dated from 2003
through 2007. They are runtime records, not configuration or content that a new
Plan 9 HostFS instance needs in order to operate.

Their filenames still belong in the runtime manifest, but their historical
contents must not become seed data for every fresh checkout. The pre-boundary
Git history preserves those imported records for provenance. Once this design
is implemented, newly created `mail`, `runq`, and `smtp` logs begin empty just
like the other generated logs. This removal of historical runtime data from
the current tree is an explicit part of the proposed boundary, not an
accidental side effect of an ignore rule.

### The CPU service writes the listener log

The standalone drawterm prototype booted `9k10cpuf`, ran `cpurc`, and started
`aux/listen`. Normal startup appended TCP and IL listener records to
`root-fs/sys/log/listen`. No password or NVRAM content was involved, but the
tracked file changed on every tested CPU-server boot.

The generated lines were removed after the prototype, restoring the file to
its tracked empty state. That cleanup is evidence for this design, not the
intended steady-state workflow.

### Ignore rules do not apply to tracked files

An ignore rule prevents untracked generated files from appearing in normal Git
status. It does not suppress modifications to files already in the index.
Making `/sys/log` a runtime boundary therefore requires both untracking the
placeholder files and creating them when the runtime needs them.

### Runtime logs and credentials have different boundaries

Service logs must be visible at their traditional paths inside the guest and
may usefully persist between local runs. They can remain beneath the ignored
host-backed `/sys/log` directory.

NVRAM, authentication keys, and other credential-bearing profile state must
not be exported as ordinary project content. Design 0001 places standalone
NVRAM under `${XDG_STATE_HOME:-$HOME/.local/state}/plan9-hostfs` or the
`PLAN9_HOSTFS_STATE` override. This design does not weaken that rule.

## Goals

1. Preserve the guest-visible `/sys/log/<name>` paths expected by legacy Plan
   9 programs.
2. Prevent normal guest service activity from modifying tracked files.
3. Keep the direct, live `u9fs` export of `root-fs/`.
4. Prepare required runtime paths without elevated privileges.
5. Make startup idempotent and never truncate an existing log.
6. Keep the required set of placeholder files reviewable in Git.
7. Keep credential-bearing state outside the repository.
8. Give failures a clear host-side diagnostic before QEMU starts.

## Non-goals

This design does not:

- make the HostFS checkout a transactional or immutable filesystem;
- classify every writable path in the imported Plan 9 tree;
- rotate, archive, upload, or parse guest logs;
- move credentials into `/sys/log` or another ignored checkout path;
- provide different Unix identities for different Plan 9 users;
- require an overlay filesystem, bind mount, container, or root privilege; or
- erase logs automatically when QEMU exits.

## Decision

### Generated `/sys/log` subtree

Files whose sole purpose is mutable service logging will no longer be tracked
beneath `root-fs/sys/log/`. The directory and its generated contents will be
ignored by Git.

The implementation will derive a complete initial manifest from the currently
tracked placeholder inventory before untracking it. The manifest belongs in
host project metadata, tentatively:

```text
host/runtime-log-files
```

Each nonblank, non-comment line names one guest-root-relative regular file,
for example:

```text
sys/log/listen
```

The manifest is declarative; it contains no log data and no secrets. Its exact
inventory initially includes all 32 currently tracked log names whose paths
remain valid. The three imported nonempty logs contribute their names only;
their historical records remain available from the parent Git history and are
not copied into generated files.

The manifest's exact filename may change during implementation if a broader
runtime-manifest format is justified, but the inventory must remain versioned
and reviewable.

### Launcher preparation

Before `u9fs` or QEMU starts, `host/qemu` will prepare the manifest entries
beneath the resolved `root-fs/` directory.

Preparation must:

1. validate that every manifest path is relative and remains beneath
   `root-fs/sys/log/`;
2. create the log directory if it is missing;
3. create each missing regular file with host-user ownership;
4. use ordinary private-by-default host permissions appropriate to the
   existing single-Unix-user export;
5. leave every existing file and its contents unchanged; and
6. fail before launching QEMU if a required path cannot be prepared safely.

The operation is idempotent. Repeated launches retain accumulated logs. It is
not a cleanup, reset, or migration command.

The default terminal profile and standalone drawterm profile use the same
preparation because either current or future guest startup may write standard
logs. Runtime-boundary behavior should not depend on which user-facing profile
happened to reveal it first.

### Git boundary

Implementation requires a coordinated change:

1. record the current placeholder inventory in the manifest;
2. remove those empty placeholder files from the Git index without treating
   their generated successors as project source;
3. ensure the generated `root-fs/sys/log/` subtree is ignored; and
4. teach the launcher to seed missing files before boot.

All four parts belong in one coherent implementation. Applying only the
ignore rule would leave tracked files dirty. Applying only the removal would
cause Plan 9 logging to change or fail when files are absent.

### Retention and user control

Generated logs persist in the checkout between runs so they remain available
for local diagnosis. They are disposable runtime data, not repository history.
The launcher will not remove or truncate them automatically.

A later explicit cleanup command may be designed if accumulated state becomes
burdensome. Such a command must validate its exact target, explain that the
operation is destructive, and avoid credential state. It is not part of this
milestone.

### Security and privacy

Runtime logs are not credential storage, but they can contain usernames,
service names, network addresses, errors, and activity timestamps. Ignoring
them prevents accidental ordinary commits; it does not encrypt them or make
them safe to publish.

The launcher must not print log contents during preparation. Documentation
should advise users to review runtime logs before sharing diagnostics or
archiving a checkout.

## Acceptance criteria

The runtime-state implementation is complete when it demonstrates all of the
following:

- the versioned manifest accounts for every placeholder intentionally removed
  from `root-fs/sys/log/`;
- the initial inventory accounts for all 32 tracked names, distinguishes the 29
  empty placeholders from the three historical nonempty logs, and does not seed
  the historical records into a fresh runtime;
- a fresh checkout has no generated log files before its first launch;
- `./host/qemu` creates every required missing log file before guest startup;
- standalone CPU-server startup can append to `/sys/log/listen` normally;
- repeated launches preserve existing log contents rather than truncating
  them;
- generated logs do not appear as tracked modifications or untracked noise in
  normal Git status;
- the default terminal boot still reaches Rio;
- standalone drawterm boot still reaches `gnot#` and accepts drawterm;
- failure to create a required log produces a clear error and no QEMU process;
- manifest validation rejects absolute paths and parent traversal;
- no launcher step requires root or changes files outside the repository and
  the configured external profile-state root; and
- NVRAM and other credential-bearing state remain outside `root-fs/`.

## Alternatives considered

### Keep tracking empty log files

Rejected. Service activity necessarily modifies them and makes a successful
runtime session look like a source change.

### Remove the files without seeding them

Rejected. Legacy `syslog` opens existing named files and does not create them.
Missing placeholders can redirect messages to the console or drop them.

### Add an ignore rule while leaving files tracked

Rejected. Git ignore rules do not suppress modifications to tracked paths.

### Mark files `assume-unchanged` or `skip-worktree`

Rejected. Those are local index hints, do not clone, can conceal intentional
changes, and make repository state harder to reason about.

### Reset or truncate logs after each run

Rejected. Automatic destructive cleanup can race active services, loses useful
diagnostics, and still leaves the tree dirty while QEMU is running.

### Mount an external or in-memory `/sys/log`

Deferred. A bind mount, overlay, ramfs, or guest namespace replacement could
create a stronger separation, but adds host privileges or guest boot
complexity. Manifest seeding preserves the current direct HostFS model and is
adequate for the first runtime-state boundary.

### Store all runtime state outside the checkout

Deferred as a general policy. It is mandatory for credentials, but relocating
ordinary guest logs while preserving `/sys/log` would require an additional
mount or export mechanism. The ignored generated subtree is simpler and keeps
logs directly inspectable on the host.

## Future work

Later investigation should inventory other mutable guest paths such as DNS
state, temporary files, mail queues, and per-user caches. Each category should
be classified deliberately as versioned content, ignored in-tree runtime data,
or external sensitive state. This design establishes the method for logs but
does not prejudge every path in the imported distribution.
