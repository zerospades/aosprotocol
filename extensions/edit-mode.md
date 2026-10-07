# Edit Mode

Lifts the client's building limits, for map editing and creative servers.

| ------------: | ------------- |
| Extension ID: | `0x36`        |
| Packet ID:    | `0x76`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name        | Direction         | Size       |
|--------|-------------|-------------------|------------|
| 0      | Edit Mode   | Server -> Client  | 3          |
| 1      | Place Model | Client <-> Server | 15 + model |

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
* may build and destroy as a spectator, from the camera position;
* may send Place Model.

The map bounds and the water level still apply. Edit mode lasts until the next
Edit Mode.

A spectator builds with [Block Action](../protocol075.md#block-action) and
[Block Line](../protocol075.md#block-line) under its own player id, in the colour
of its last [Set Colour](../protocol075.md#set-colour). Every client applies
them like any other player's.

## Sub ID 1: Place Model

Places a voxel model in the map. A client in edit mode sends it to the server,
and the server sends it to every client, the sender included, if it allows it.

| Field Name    | Field Type | Example | Notes                                            |
|---------------|------------|---------|--------------------------------------------------|
| Packet ID     | UByte      | `0x76`  | Always `0x76`.                                   |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.                  |
| Player ID     | UByte      | `0`     | The player who placed it. Ignored by the server. |
| X position    | LE Int     | `256`   | Where the model's voxel `0, 0, 0` goes.          |
| Y position    | LE Int     | `256`   |                                                  |
| Z position    | LE Int     | `30`    |                                                  |
| Model         | Byte[]     |         | A KV6 file, to the end of the packet.            |

The model is at most 65,536 bytes. Each of its voxels becomes a block of the
voxel's colour, replacing what was there; its empty cells leave the map as it
is. Its axes are the map's, and its pivot is ignored. Voxels outside the map or
below the water level are dropped.

A client applies the model only when the server sends it. The server may drop it
or change its position.

The server still checks every block and model, and drops those it does not
allow.

See [Extensions](extension.md) for how the extension is negotiated.
