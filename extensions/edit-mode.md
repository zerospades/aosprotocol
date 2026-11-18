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

In edit mode the player:

* never runs out of blocks, and a [Block Line](../protocol075.md#block-line) may
  be any length;
* may build and destroy at any distance they can aim at;
* has no delay between two builds or two spade hits;
* may build and destroy as a spectator, from the camera position.

The map bounds, the water level and the need for a neighbouring block still
apply. The server still checks every block, and drops those it does not allow.
Edit mode lasts until the next Edit Mode.

A spectator builds with [Block Action](../protocol075.md#block-action) and
[Block Line](../protocol075.md#block-line) under its own player id, in the colour
of its last [Set Colour](../protocol075.md#set-colour), which the server
relays. Every client applies them like any other player's.

A client draws at most 50 blocks of a received Block Line. The server sends a
longer line to the other clients as [Block Action](../protocol075.md#block-action)
builds, one per block of the line.

See [Extensions](extension.md) for how the extension is negotiated.
