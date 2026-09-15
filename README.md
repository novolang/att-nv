# att-nv

The Attribute Protocol (ATT) is how one Bluetooth device reads and
writes the values another one publishes. It is specified in the
[Bluetooth Core Specification](https://www.bluetooth.com/specifications/specs/core-specification/),
Volume 3, Part F. This package brings it to novo-lang with no radio and
no server loop around it, together with the Generic Attribute Profile
(GATT) declarations of Volume 3, Part G that give those values their
meaning. It runs on a channel that
[l2cap-nv](https://novo-lang.org/packages/l2cap-nv) provides and depends
on that package for it.
[smp-nv](https://novo-lang.org/packages/smp-nv) is its sibling, the
Security Manager, on the next channel along.

**Status: NOT IMPLEMENTED — interface only.** Every function is declared
with its full signature, but every body is a `todo()` that panics when
called. The package is published so its design can be reviewed and
depended on before it is implemented. Version 0.1.0 will be the first
working release.

## What the Attribute Protocol is

A device that has something to publish holds a table. Each row is an
**attribute**: a 16-bit **handle** that names it, a **type** that says
what it is, a value, and the permissions a server enforces before
letting a peer near it. The type is a **UUID**, either two bytes from
the Bluetooth assigned-numbers list or a full sixteen. Handle 0x0000 is
reserved, so a real attribute's handle is 0x0001 or above.

The protocol over that table is twenty-odd request and response pairs on
one L2CAP channel. A **client** sends a request and the **server**
answers it, or answers an **Error Response** saying why not. Two further
kinds of message exist. A **command** expects no answer at all. A
**notification** is the server pushing a value out unacknowledged, and
an **indication** is the same push with a confirmation owed back.

Every message starts with one **opcode** byte. Its low six bits are the
**method**, which says which of the twenty-odd messages it is. The top
two bits are flags: 0x40 marks a command and 0x80 marks a message
carrying a twelve-byte authentication signature. Section 3.3.1 defines
the layout.

| Quantity | Value |
| --- | --- |
| The channel ATT runs on | CID 0x0004 |
| A handle | 16 bits, 0x0000 reserved |
| A UUID | 2 bytes or 16 bytes |
| The opcode's method | The low 6 bits |
| The command flag | 0x40 |
| The authentication-signature flag | 0x80 |
| An authentication signature | 12 bytes |
| The default and minimum MTU | 23 bytes |

The **maximum transmission unit** (MTU) is the largest PDU the two peers
have agreed to exchange. Every link starts at 23 bytes, which is the
27-byte link-layer payload less the four-byte L2CAP header, so a 23-byte
ATT PDU is the largest one that never needs fragmenting. The Exchange
MTU request and response each offer a number, and the smaller of the two
is what the link uses (section 3.4.2).

GATT is not a second protocol. It is a set of rules about what the
values in the table mean. An attribute of type 0x2800 declares a
**service**, a group of related attributes. One of type 0x2803 declares
a **characteristic** and says which handle holds its value and what may
be done with it. A client discovers a whole profile by reading those
with ordinary ATT requests. Volume 3, Part G section 3 assigns the
numbers.

| Type | Declares |
| --- | --- |
| 0x2800 | A primary service |
| 0x2801 | A secondary service, one only another service includes |
| 0x2802 | An include, one service inside another |
| 0x2803 | A characteristic, with its value handle and properties |
| 0x2900 | The Characteristic Extended Properties descriptor |
| 0x2901 | The Characteristic User Description descriptor |
| 0x2902 | The Client Characteristic Configuration descriptor, which a client writes to turn notifications on |
| 0x2904 | The Characteristic Presentation Format descriptor |

This package performs no input or output, and it keeps nothing between
requests. The table is a value the caller holds, and every discovery
request is a range scan over it, so the same walk runs over a database
in flash, one built at boot and one a test wrote down.

## Install

```
novo pkg add att-nv
```

## Example

```novo
use att

fn main() [io]
    // What a client may do with an attribute, and what it must have done
    // first. None of this goes on the wire.
    let readable = AttPermissions {
        readable: true, writable: false,
        encryption_required: false, authentication_required: false,
        authorization_required: false, minimum_key_size: 0 }

    // One row of a server's table: handle 0x0003 holds the manufacturer
    // name, whose assigned type number is 0x2A29.
    let row = AttAttribute {
        handle: 0x0003,
        attribute_type: AttUuid16(0x2A29),
        value: [0x6E as u8, 0x6F as u8, 0x76 as u8, 0x6F as u8],
        end_group_handle: 0x0003,
        permissions: readable }

    // A Read Request for that handle, as it arrives on CID 0x0004.
    let f = L2capFrame { cid: att.CID, payload: [0x0A as u8, 0x03 as u8, 0x00 as u8] }

    match att.database([row])
        Err(e) => println("these rows are not a table: ${e.message()}")
        Ok(db) =>
            match att.from_frame(f)
                Err(e)  => println("not an ATT pdu: ${e.message()}")
                Ok(pdu) =>
                    match pdu
                        ReadRequest(handle) =>
                            // The walk answers with the PDU the peer is
                            // owed, which is an Error Response when the
                            // handle is not in the table.
                            let answer = att.read(db, handle, att.MTU_DEFAULT)
                            println("${list.len(att.encode(answer))} bytes go back")
                        _ => println("some other request")
```

Build and test with `novo pkg build` and `novo test`. Today `novo test`
fails on purpose: every test reaches a `not implemented: att.<fn>`
panic. The tests are the specification the implementation will have to
satisfy.

## What the package contains

| Module | Contents |
| --- | --- |
| `att` | The protocol. Every ATT PDU in both directions as one type, encoding and decoding, the opcode arithmetic, the MTU rule, the attribute table and the permissions, and the walk that answers each discovery and access request. |
| `gatt` | The declarations ATT reads. The assigned type numbers, the characteristic properties, services, characteristics and descriptors as values, the bytes each way, and the functions that turn a profile into the rows a table holds. |

## How to choose an entry point

**`encode` and `decode` are the codec.** They turn a PDU into the bytes
of an L2CAP payload and back. Use them in a capture tool, in a test, and
anywhere the channel is already handled.

**`frame` and `from_frame` are the same thing with the channel
attached.** They speak l2cap-nv's `L2capFrame`, so a frame that arrived
on the wrong channel is refused here rather than parsed as an opcode.
Use them in a stack built on l2cap-nv.

**The walks are the server.** `find_information`, `find_by_type_value`,
`read_by_type`, `read_by_group_type`, `read`, `read_blob`,
`read_multiple` and `write` each take a request's fields and the MTU and
return the PDU that answers it. Use them when you are the server. A
client uses `encode` to ask and `decode` to read the answer.

**`gatt` is for whoever writes the profile.** `service_attribute`,
`characteristic_attribute`, `characteristic_value_attribute` and
`descriptor_attribute` turn services and characteristics into the rows
`att.database` takes, so a profile is written once rather than written
and then laid out by hand.

## The rules a user needs

1. **Both handles in a range request are inclusive.** A request for
   0x0001 to 0x0005 covers five attributes, and getting the ends wrong
   is the classic ATT server bug. `attributes_in_range` is that rule on
   its own (Core Vol 3 Part F section 3.4.3.1).
2. **A walk answers with a PDU, never a `Result`.** A read of a handle
   that is not there is an Error Response, which the peer is owed and
   which the specification spells out to the byte. Send what you are
   given.
3. **`AttDecodeError` and `AttErrorCode` are different answers.** The
   first is why this package could not parse what arrived. The second is
   what goes back on the air. A three-byte Read Request produces both.
4. **The MTU is the smaller of the two offers.** `negotiated_mtu` takes
   the client's and the server's numbers from the Exchange MTU pair. A
   peer that asks for less than 23 gets 23, because a smaller link
   cannot carry a read response (section 3.4.2).
5. **Every pair in a Read By Type response is the same length.** The
   walk stops at the first attribute whose value length differs from the
   first one's. A server that checked only the MTU sends a response no
   client can split (section 3.4.4.1).
6. **A Find Information response is in one format throughout.** A table
   that mixes 16-bit and 128-bit types stops at the first change of
   width (section 3.4.3.2).
7. **Only a grouping type is legal in Read By Group Type.** That is
   0x2800 and 0x2801. Anything else is answered with
   `AttUnsupportedGroupType` (section 3.4.4.9).
8. **A Read Blob offset past the end is `AttInvalidOffset`.** An offset
   exactly at the end is an empty Read Blob Response, which is how a
   client learns it has the whole value (section 3.4.4.5).
9. **A Read Multiple response carries no lengths.** A client that does
   not already know each value's length cannot split the answer. That is
   the specification's design and this package reproduces it (section
   3.4.4.7).
10. **Compare UUIDs with `uuid_eq`, never byte for byte.** A 16-bit UUID
    and the 128-bit expansion of the same assigned number are the same
    type, and a client is allowed to spell its filter out in full.
11. **The security state is yours to supply.** This package cannot know
    whether a link is encrypted or a peer authenticated, so
    `permission_error` and `write` take those facts as arguments along
    with the encryption key size (section 3.2.5).
12. **A write of a different length is refused.** `write` answers
    `AttInvalidAttributeValueLength` rather than changing an attribute's
    width, because a fixed-width characteristic that silently changed
    size is how a client's next read stops parsing.
13. **One request at a time per link.** A client that sends a second
    request before the first is answered has broken the protocol. The
    rule needs state that outlives a request, so it is yours to enforce;
    `is_request` is the predicate it is written against (section 3.3.2).
14. **A reserved bit in a Client Characteristic Configuration write is
    reported, not masked.** `read_client_configuration` refuses it, and
    the caller answers the peer `AttValueNotAllowed` (Core Vol 3 Part G
    section 3.3.3.3).

## Running on a microcontroller

novo-lang lets a package state which of its modules can run on a device
with no heap allocator, and the compiler checks that claim on every
build. Here the claim covers both modules, and it matters more here than
anywhere else in this family: the device on a coin cell is usually the
server.

```bash
novo build --target=nrf52-qemu src/main.nv
```

`tests/embedded_probe.nv` is that claim as a program that either builds
or does not. It builds, producing a Cortex-M4 executable from the probe,
this package, l2cap-nv and hci-codec-nv.

A device cannot write a UUID down. `AttUuid` has two variants because
the wire distinguishes the two widths everywhere, and at the embedded
tier a payload-carrying variant is a heap cell, so `AttUuid16(0x2800)`
cannot be constructed there. A device can receive a UUID, match on one
and pass one along, which is what a server does with the type in a
request. The probe takes the type it searches for from an argument for
that reason. This is an open toolchain defect rather than a property of
the protocol, and the signatures are published unchanged.

The same tier rules mean a probe cannot write its own table down: a list
literal and a boxed struct literal are both allocations. On a device it
does not have to. The rows arrive from flash and the requests arrive
from the connection.

## What is not included

- **A server loop.** Dispatch, the prepared-write queue behind Prepare
  Write and Execute Write, notification subscriptions, and the
  one-request-at-a-time rule. Each needs state that outlives a request,
  and a package that held it would be holding a connection.
- **The Read Multiple Variable pair, 0x20 and 0x21, and Multiple Handle
  Value Notification, 0x23.** All three are Bluetooth 5.2's.
  `AttHandleValue` is declared for them and nothing decodes into it yet.
- **Signature verification.** A Signed Write Command carries its twelve
  signature bytes and this package hands them over. Checking them is a
  connection-signature-resolving-key operation that belongs beside
  smp-nv's cryptographic toolbox rather than in a parser.
- **The permissions policy.** GATT says what a characteristic may be
  used for and says nothing about what security it needs.
  `AttPermissions` is a value the caller fills in, and a package that
  guessed would be inventing a policy.
- **A channel and a radio.** PDUs arrive as bytes and leave as bytes.
  Getting them to and from the peer is l2cap-nv's work and the
  controller's.

## Related packages

- [l2cap-nv](https://novo-lang.org/packages/l2cap-nv) is the layer
  below. It provides the channel ATT runs on, and this package depends
  on it so that `frame` and `from_frame` speak its `L2capFrame` rather
  than a second spelling of a channel and a payload.
- [smp-nv](https://novo-lang.org/packages/smp-nv) is the Security
  Manager, on CID 0x0006. It is what makes a link encrypted and
  authenticated, which is the state `permission_error` is told about.
  Pair with smp-nv, then read with this package.
- [hci-codec-nv](https://novo-lang.org/packages/hci-codec-nv) is two
  layers below, the interface to a controller.
- [ble-link-codec-nv](https://novo-lang.org/packages/ble-link-codec-nv)
  is the link layer, for a program that drives a radio itself.
- The standard library has no Bluetooth module. `std.net` is sockets and
  has nothing to do with attributes.

## Tests

```bash
novo test tests/att_tests.nv      # 32 tests against the signatures
novo test tests/gatt_tests.nv     # 12 tests over the declarations
```

The byte strings the suite asserts against are the Bluetooth Core
Specification's own, Volume 3 Part F section 3.4: the Exchange MTU pair,
the Error Response layout, and the Read By Group Type response that
reports one primary service over handles 0x0001 to 0x0005. The database
they walk is a five-row Device Information service, which is the
smallest table that exercises every walk: a grouping attribute with an
end-group handle, two characteristic declarations, and two values of
different lengths. The GATT suite checks the declarations of Volume 3
Part G section 3 against the bytes a response carries.

The tests compile today and fail at run, each on the `not implemented`
panic that is its body. That is the expected state of an interface
release. They turn green one at a time as bodies land.

## Implementation status

| Item | Implemented |
| --- | --- |
| `att.CID`, `.MTU_DEFAULT`, `.MTU_MINIMUM`; every `gatt.UUID_*` | yes (they are constants) |
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

## Licence

Apache-2.0. See `LICENSE`.

<!-- docs/writing-a-readme.md is the style guide for this page. -->
