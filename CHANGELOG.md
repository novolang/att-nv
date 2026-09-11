# Changelog

Every published version, newest first. This file is on the publish
allow-list, so it travels with the package: it is the only thing a
consumer deciding whether to upgrade can read.

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
