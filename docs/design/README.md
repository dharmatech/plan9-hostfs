# Design documents

This directory records architectural decisions for Plan 9 HostFS.

## Status values

- **Draft**: under investigation or awaiting end-to-end verification.
- **Accepted**: approved for implementation.
- **Superseded**: retained for history but replaced by a later design.

## Index

| Design | Status | Summary |
| --- | --- | --- |
| [0001: Drawterm access](0001-drawterm-access.md) | Accepted | Opt-in standalone CPU service, persistent private NVRAM, and loopback-only drawterm forwarding. |
| [0002: Repository layout](0002-repository-layout.md) | Accepted | Separate host tooling and project metadata from the Plan 9 root exported through u9fs and TFTP. |
| [0003: HostFS runtime state](0003-hostfs-runtime-state.md) | Accepted | Keep mutable guest runtime logs out of tracked source while preserving the live HostFS model. |

## Process

Design documents describe both the intended behavior and the evidence behind
it. Source-supported expectations remain distinguishable from behavior that
has been verified by booting the guest.

A design may be accepted before every future enhancement is settled, but its
acceptance criteria must identify what is required for its implementation
milestone. Later discoveries should update the relevant design or add a new
numbered document when they introduce a separate architectural decision.

The documents are project history. They remain in the repository after
implementation so later changes can be evaluated against the original
constraints and tradeoffs.
