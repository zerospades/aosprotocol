# Extensions

Extensions allow adding new packets to the 0.75 protocol as well as querying the
support for client and server functionality. This page describes how extensions
are identified and negotiated; each extension itself is specified in its own file
in this folder, and the registry of ids lives in the
[0.75 documentation](../protocol075.md#extensions).

## Extension IDs

Each extension is given an unique id that is decided when it is first
registered. We differentiate between two types of packets:

*Those that:*

| Type          | Purpose                               | Extension id range |
| ------------- | ------------------------------------- | ------------------ |
| `HAS_PACKETS` | introduce new packets to the protocol | 0-191              |
| `PACKETLESS`  | don't use and need any packets        | 192-255            |

Each extension is given one legacy packet id equal to `64+extension_id`.
For `PACKETLESS` extensions this would mean that their packet ids are out
of spec `>255`, thus they don't have any.

An example for a packetless extension would be *OpenSpades'* UnicodeExt.

Packetful extensions are extensions that can be used to send packets. Each
extension has 256 packet types reserved for itself. This is useful for adding
functionality that requires sending additional information from or to the client.

Packetless extensions are extensions that can not send any packets. This is useful
for signalling support for additional values in or certain behaviours related to
existing packets.

Packetless extensions exist as an artefact of the implementation of extensions.
As the space reserved for extension packets is limited, values above 191 do not
have any packet types left.

Some extensions have no registered extension id. They are not announced through
the `ExtInfo` packet.

## Extension Packets

Each extension packet will contain 1 additional byte in its data, which is a
subpacket id, used to have multiple packets available for each extension. This
is always the case, even if an extension only needs 1 packet in total.

General extension packet structure:

| Field name    | Type      | Notes          |
| ------------- | --------- | -------------- |
| Packet id     | UByte     | 64-255         |
| Sub packet id | UByte     | 0-255          |
| Data          | UByte[]   | extension data |

## ExtInfo Packet

| ----------: | ------------------ |
| Packet ID   | 60                 |
| Total size: | `2+2*length`       |

| Field name | Field type   | Notes                        |
| ---------- | ------------ | ---------------------------- |
| length     | UByte        | `length` entries will follow |
| entries    | ExtInfoEntry | see below                    |

**ExtInfoEntry**

| Field name | Field type | Notes                        |
| ---------- | ---------- | ---------------------------- |
| ext. ID    | UByte      | see [Extension IDs](#extension-ids) |
| version    | UByte      | Usually starts at 1          |

## Negotiation Flow

The server should send an `ExtInfo` packet (optimally) after the Version Info response has been received to compatible clients
(OpenSpades versions > 0.1.3, see https://github.com/piqueserver/piqueserver/issues/504),
assuming it supports any. The client can store the list of extensions for later use and should
reply with an `ExtInfo` packet that lists the extensions it supports (if it does actually support any).

The client can omit any extensions that the server does not support from its
reply, but this is not necessary as the server can simply ignore them itself.
