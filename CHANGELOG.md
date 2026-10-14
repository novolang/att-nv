# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

## 0.1.0 — 2026-10-10

The **implementation**.  Every body the 0.0.2 interface published is
written, `novo test` is green over three suites, and every line under
`src/` is covered by them except one guard, excused with a `cov: skip`
marker.

- **A refusal with no width is `AttRaggedList(unit: 0)`.**  Four faults
  have no record width to report — an unknown Find Information format
  byte, an unknown Execute Write flag, a reserved bit in a Client
  Characteristic Configuration write, and two rows at one handle in
  `database` — and each answers the value it could not read in
  `actual`.  A variant per fault would put a parse detail nobody can act
  on in the published error type.
- **`error_code_of` is total over all 256 bytes.**  The named codes are
  the specification's; the two ranges that belong to somebody else keep
  their byte as `AttApplicationError` (0x80..0x9F) and
  `AttCommonProfileError` (0xE0..0xFF); everything above 0x13 that is
  neither is `AttReservedError`, so a peer that invents a code is
  reported rather than dropped.  `error_code_value(error_code_of(b))`
  is `b` for every byte, which the edge suite asserts.
- **A record list is one width, and a walk stops at a change.**  A pair,
  a triplet and a Find Information pair each have one shape per
  response, so a walk that meets another width answers what it has and
  stops; mixing widths would build a response no client can split
  (sections 3.4.3.2, 3.4.4.1).
- **A record's length is one byte, so a pair's value stops at 253 and a
  triplet's at 251.**  A longer value is cut to the cap rather than
  refused, which leaves a Read Blob to finish with and is what the
  one-byte length field can hold once the handles are counted.
- **A read is cut to the MTU; a Read Multiple value is not cut at
  all.**  `read` answers the part that fits and the client reads the
  rest from an offset.  A Read Multiple response carries no lengths, so
  the run ends at the first value that would not fit whole rather than
  letting a client split it at the wrong byte.
- **A Read Blob offset exactly at the end is an empty response.**  Past
  the end is `AttInvalidOffset`; at the end it is the empty part that
  tells a client it has the whole value.
- **An empty answer is an Error Response.**  A walk that matched no row
  answers `AttAttributeNotFound` for the opcode it was asked, because
  the peer is owed an answer either way and an empty response does not
  say the range is over.
- **A write of a different length does not change the row's width.**
  It is answered `AttInvalidAttributeValueLength` with the database
  unchanged, because a fixed-width characteristic that silently changed
  size is how a client's next read stops parsing.  The permissions are
  checked first, through the same `permission_error` a caller can call
  on its own.
- **The MTU floor is a number, not a fault.**  `data_room` answers zero
  for an MTU below a response's fixed part rather than a negative size,
  so a walk at 23 bytes, or at 0, still answers the PDU it can fill.

Four bodies bind a `match` to a `let` and return it afterwards —
`att.decode`'s type-naming arm, `AttDecodeError.message`, and `gatt`'s
two declaration readers.  A statement-position `match` whose arms all
`return`, inside a `system` block, does not type-check on novo 0.20.2:
it answers `cannot infer the type of this expression (resolved to
`_`)` (E2000), so the value is bound first and returned once.  The
all-`return` match as a match arm's body, the other shape filed from
this family, does not appear at all; the arm that would have needed it
answers an `if`/`else` instead.

The new tests live in `tests/att_edge_tests.nv` — the round trip of all
twenty-eight PDUs at every opcode, the sentence each `AttDecodeError`
answers, the width changes and the value caps a walk stops at, the
permission rule one requirement at a time, and a profile written with
`gatt` and then walked with `att` — because the two API suites are the
published oracle and are not edited.

The embedded probe is built from `src/`, not from where it sits.  A
`novo build` of a file that is not a package's `main` resolves the
package's `src/` siblings but does not read `[dependencies]`, so
`novo build --target=nrf52-qemu tests/embedded_probe.nv` fails on the
names l2cap-nv declares; a board build of a file directly under `src/`,
which is the route the shard audit takes, assembles the dependency and
produces the ELF.

`novo` is required at `>= 0.20.2`, the version this implementation is
written and verified against.  The 0.8.9 floor was true of the interface
— `todo()` bodies build anywhere — and false of the implementation, and
a floor set too low fails as a wrong answer rather than a build error.

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
