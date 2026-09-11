# att-nv

**Status: NOT IMPLEMENTED — interface only.**

Every public function below is published with its signature and its
effect row, and every body is `todo()`.  Installing this package works;
calling it panics with `not implemented`.

## What this is

The Bluetooth Attribute Protocol with no radio and no server loop
around it (Core Vol 3 Part F): every ATT PDU as a typed value in both
directions, the MTU exchange, and the attribute-database walk a GATT
server performs — find-by-type, read-by-type, read-by-group-type, the
handle ranges — as arithmetic over a database the caller holds.  Beside
it, `gatt` carries the declarations ATT reads (Core Vol 3 Part G): the
service and characteristic declarations as values, and the two
functions each way between them and the bytes a response carries.

It sits above l2cap-nv and speaks its vocabulary: `frame` produces an
`L2capFrame` on CID 0x0004 and `from_frame` refuses one on any other
channel, so the two packages share one spelling of a channel and a
payload.

## Adding it, and checking it

```bash
novo pkg add att-nv            # into your novo.toml
novo pkg build                 # type- and effect-check the package
novo test tests/att_tests.nv
novo test tests/gatt_tests.nv
```

`novo test` is red today and that is the point of the release: every
assertion below the first constants fails with `not implemented:
att.<fn>`.  They turn green one at a time as bodies land.

## The one example that will work

```novo
use att
use gatt

// A server's whole request path: a frame arrives, a PDU comes back.
// There is nothing between them that this package owns.
fn on_frame(db: AttDatabase, mtu: Int, f: L2capFrame) -> ?L2capFrame
    match att.from_frame(f)
        Err(_)  => None
        Ok(pdu) =>
            match pdu
                ExchangeMtuRequest(_)              => Some(att.frame(ExchangeMtuResponse(mtu)))
                ReadRequest(handle)                => Some(att.frame(att.read(db, handle, mtu)))
                ReadByGroupTypeRequest(s, e, kind) =>
                    Some(att.frame(att.read_by_group_type(db, s, e, kind, mtu)))
                _                                  => None
```

## The layer, and why

`core`.  Every walk is a range scan over a list the caller owns, every
PDU is bytes in and bytes out, and the server keeps nothing between
requests — the prepared-write queue, the notification subscriptions and
the outstanding-request rule all need state that outlives a request, so
they are the caller's and this package does not pretend otherwise.

`tests/embedded_probe.nv` is that claim in a form that either builds or
does not, and **it builds**: `novo build --target=nrf52-qemu` produces
a Cortex-M4 ELF from the probe, this package, l2cap-nv and
hci-codec-nv.  The claim matters more here than anywhere else in the
split — the device on a coin cell is the SERVER, so a GATT server that
could not compile for one would have its whole audience on the other
side of the link.

The shard audit's `core-embedded` row reports **pass** on this package,
and that pass is worth less than it looks.  The row assembles its
scratch package with an empty `[dependencies]`, so l2cap-nv is not
there — and this probe reaches it only as a type in a signature
(`att.from_frame(frame: L2capFrame)`), which resolves to nothing
quietly.  The same scratch build links with no copy of l2cap-nv on the
machine at all.  l2cap-nv's own probe fails the same row, because it
names a CONSTRUCTOR from its dependency rather than a type.

Both halves are filed against the audit rather than designed around.
The build that actually checks this package's device claim is the
hand-linked one above, with every dependency present.

## The load-bearing interface

Two decisions.  The first is that the walks return a **PDU**, not a
`Result`:

```novo
pub fn read(db: AttDatabase, handle: Int, mtu: Int) -> AttPdu
pub fn read_by_group_type(db: AttDatabase, starting_handle: Int, ending_handle: Int,
                          group_type: AttUuid, mtu: Int) -> AttPdu
```

A read of a handle that is not there is not an error in this package's
sense — it is an **Error Response**, which the peer is owed and which
the specification spells out to the byte.  A `Result` would make every
caller turn a refusal into that PDU itself, which is writing the
protocol a second time and getting the request-opcode field wrong once.

The second is that the list-shaped responses carry `data` with a record
width beside it rather than a list of structs:

```novo
    ReadByTypeResponse(pair_length: Int, data: [u8])
    ReadByGroupTypeResponse(triplet_length: Int, data: [u8])
```

That is the wire format, and it is also the only shape that lets a
server fill a buffer to the MTU and stop.  A list of structs has to be
built whole and then measured, and on a device the whole is what does
not fit.

And the third thing worth saying is what is NOT one type:
`AttDecodeError` is why a PDU could not be parsed here, `AttErrorCode`
is what goes back to the peer.  A client that sends a two-byte Read
Request has done both — sent something unparseable and earned an
Invalid PDU response — and one enum for both would lose either the
length that was wrong or put a parse detail on the air.

## Where the no-allocation shape ran out

A UUID is two bytes or sixteen, and the wire distinguishes them
everywhere, so `AttUuid` is an enum with a payload.  **At
`@tier(embedded)` a payload-carrying variant is a heap cell and cannot
be constructed**, so a device cannot write `AttUuid16(0x2800)` down at
all: it can receive one, match on one and pass one along, and it cannot
make one.  The probe takes the type it discovers from an argument for
that reason.

Nothing here is designed around it.  The alternative — a fixed-width
`@value` struct holding a `[u8; 16]` and a width flag — is worse for a
different reason: a `@value` struct may not be the payload of a
`Result`, an optional, an enum or a boxed struct (E2015), so it could
not be a field of `AttAttribute` and could not be returned by a
decode that can fail.  Both are open toolchain defects, filed from this
lane; the signatures stay as they are, because a package that changed
its types to fit a compiler limitation would be publishing the
limitation as a design.

## What is not here, and what a consumer should expect

**A server loop.**  Dispatch, the prepared-write queue behind Prepare
Write and Execute Write, notification subscriptions, and the one-
request-at-a-time rule.  Each needs state that outlives a request, and
`is_request` is the predicate a caller enforces the last of those with.

**The Read Multiple Variable pair** (0x20 / 0x21) and **Multiple Handle
Value Notification** (0x23), which are Bluetooth 5.2's.
`AttHandleValue` is declared for them and nothing decodes into it yet.

**Signature verification** for Signed Write Command: the PDU carries
its twelve signature bytes, and checking them is a CSRK operation that
belongs beside smp-nv's toolbox rather than in a parser.

## The reference implementation

`orbit/ble`'s ATT layer, spread across four modules that this package
makes two: `host/att.nv` (867 lines — the opcode constants, the error
codes, and a builder and a reader per PDU field),
`host/att_server.nv` (1,294 — the walks over a heap database blob),
`host/att_server_scratch.nv` (1,014 — the same walks again, written
against fixed RAM) and `host/gatt_db.nv` (549 — one profile, written as
constants).

Three things change in the port, and each is a thing the reference could
not have.  The forty-odd `read_*_req_*` and `build_*_rsp` functions
become `decode` and `encode` over one `AttPdu`, so a field cannot be
read off the wrong PDU.  The `ATT_ERR_*` integers become
`AttErrorCode`, so an error code cannot be compared against an opcode.
And the database stops being a blob at a fixed address: every
`*_to_scratch*` function in `att_server_scratch.nv` carries a `[hw]`
row for no reason except that it writes to 0x2000F700, and the same
arithmetic over a caller-held value carries none.

The Bluetooth Core Specification Vol 3 Parts F and G are the source of
the test vectors.

## Status

| item | implemented |
| --- | --- |
| `att.CID`, `.MTU_DEFAULT`, `.MTU_MINIMUM`; every `gatt.UUID_*` | yes — they are constants |
| `att.negotiated_mtu` | no |
| `att.encode_uuid`, `.decode_uuid`, `.uuid_eq` | no |
| `att.error_code_value`, `.error_code_of` | no |
| `att.encode`, `.decode`, `.opcode`, `.opcode_method`, `.opcode_is_command`, `.opcode_is_signed`, `.is_request` | no |
| `att.frame`, `.from_frame` | no |
| `att.database`, `.attribute_count`, `.attribute_at`, `.handle_range`, `.attributes_in_range` | no |
| `att.find_information`, `.find_by_type_value`, `.read_by_type`, `.read_by_group_type` | no |
| `att.read`, `.read_blob`, `.read_multiple` | no |
| `att.permission_error`, `.write` | no |
| `att.AttDecodeError.message` | no |
| `gatt.properties_byte`, `.properties_of` | no |
| `gatt.service_declaration_value`, `.characteristic_declaration_value`, `.client_configuration_value` | no |
| `gatt.read_characteristic_declaration`, `.read_service_declaration`, `.read_client_configuration` | no |
| `gatt.service_attribute`, `.characteristic_attribute`, `.characteristic_value_attribute`, `.descriptor_attribute` | no |
