# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.0.2 — 2026-09-15

README rewritten to the package README style guide (docs/writing-a-readme.md); no change to the interface.

## 0.0.1 — 2026-09-11

The **interface**, before anyone implements it. Every signature, every
type and every effect row is published; every body is `todo()`, and the
release is stamped `NOT IMPLEMENTED — interface only`. Adding this
package works and calling it panics.

- `AttPdu` — every ATT PDU, both directions, as one enum. The
  list-shaped responses carry their records as bytes with the record
  width beside them, which is the wire format and the only shape a
  server can fill to the MTU and stop.
- `AttErrorCode` and `AttDecodeError` as two types: what goes back to
  the peer, and why this package could not parse what arrived.
- `AttDatabase` and the walks over it — find-by-type, read-by-type,
  read-by-group-type, read, read-blob — each returning the PDU that
  answers the request, including the Error Response, because the peer
  is owed an answer either way.
- `AttPermissions` and `permission_error`: the security state arrives as
  arguments, because a core package cannot know whether a link is
  encrypted.
- `gatt` — the service and characteristic declarations as values, the
  bytes each way, and the three functions that turn a profile into the
  rows a database holds.
- `frame` and `from_frame` over l2cap-nv's `L2capFrame`, so a frame on
  the wrong CID is refused here rather than parsed as an opcode.

Two toolchain defects travel with the release rather than being designed
around, and both are about spelling a UUID without allocating: a
payload-carrying enum variant cannot be constructed at
`@tier(embedded)`, and a `@value` struct may not be the payload of a
`Result`, an optional, an enum or a boxed struct. The README says what
each costs.

### Design notes

Moved here from the README, which now states only what a user needs.

- The alternative to `AttUuid` being an enum with a payload is a
  fixed-width `@value` struct holding a `[u8; 16]` and a width flag,
  and it is worse for a different reason: a `@value` struct may not be
  the payload of a `Result`, an optional, an enum or a boxed struct
  (E2015), so it could not be a field of `AttAttribute` and could not
  be returned by a decode that can fail. Both are open toolchain
  defects, filed from this lane. The signatures stay as they are,
  because a package that changed its types to fit a compiler
  limitation would be publishing the limitation as a design.
- The shard audit's `core-embedded` row reports pass on this package,
  and that pass is worth less than it looks. The row assembles its
  scratch package with an empty `[dependencies]`, so l2cap-nv is not
  there, and the probe reaches it only as a type in a signature
  (`att.from_frame(frame: L2capFrame)`), which resolves to nothing
  quietly. The same scratch build links with no copy of l2cap-nv on
  the machine at all. l2cap-nv's own probe fails the same row, because
  it names a constructor from its dependency rather than a type. Both
  halves are filed against the audit. The build that actually checks
  this package's device claim is the hand-linked one, with every
  dependency present.
- The port collapses four modules of `orbit/ble` into two:
  `host/att.nv` (867 lines — the opcode constants, the error codes,
  and a builder and a reader per PDU field), `host/att_server.nv`
  (1,294 — the walks over a heap database blob),
  `host/att_server_scratch.nv` (1,014 — the same walks again, written
  against fixed RAM) and `host/gatt_db.nv` (549 — one profile, written
  as constants).
- Three things change in the port. The forty-odd `read_*_req_*` and
  `build_*_rsp` functions become `decode` and `encode` over one
  `AttPdu`, so a field cannot be read off the wrong PDU. The
  `ATT_ERR_*` integers become `AttErrorCode`, so an error code cannot
  be compared against an opcode. And the database stops being a blob at
  a fixed address: every `*_to_scratch*` function in
  `att_server_scratch.nv` carries a `[hw]` row for no reason except
  that it writes to 0x2000F700, and the same arithmetic over a
  caller-held value carries none.
