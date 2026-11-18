# Edit Mode

Lifts the client's building limits, for map editing and creative servers.

| ------------: | ------------- |
| Extension ID: | `0x36`        |
| Packet ID:    | `0x76`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name      | Direction        | Size |
|--------|-----------|------------------|------|
| 0      | Edit Mode | Server -> Client | 3    |

## Sub ID 0: Edit Mode

Turns edit mode on or off for the receiving client's own player.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x76`  | Always `0x76`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| State         | UByte      | `1`     | `0` off, anything else on.      |

Edit mode is off when a client joins, and changes only with the next Edit Mode.

In edit mode the player:

* never runs out of blocks, and a [Block Line](../protocol075.md#block-line) may
  be any length;
* may build and destroy at any distance;
* may build at z `62`, on the water;
* has no delay between two builds or two spade hits;
* may build and destroy as a spectator.

The client offers these to the player, and the server accepts them. A block
still needs a solid block on one of its six faces, z `63` cannot change, and the
server still drops any block it does not allow.

The client may also let the player leave their body and fly the camera freely,
and return to it at any time. This is client side only: the server sees the
player where the body is, and the player builds and destroys from the camera.

A spectator builds with [Block Action](../protocol075.md#block-action) build,
bullet destroy and spade destroy, and with
[Block Line](../protocol075.md#block-line), under its own player id. Its blocks
take its colour from [Existing Player](../protocol075.md#existing-player), then
from each [Set Colour](../protocol075.md#set-colour) it sends, which the server
relays. Every client applies them like any other player's.

A client that negotiated Edit Mode draws a received Block Line in full, whatever
its length. Other clients may draw only part of a long line, so the server sends
them a longer line as Block Action builds, one per block of the line.

See [Extensions](extension.md) for how the extension is negotiated.
