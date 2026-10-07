# Edit Mode

Lifts the client's building limits, for map editing and creative servers.

| ------------: | ------------- |
| Extension ID: | `0x36`        |
| Packet ID:    | `0x76`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

## Sub ID 0: Edit Mode

Server to client only. Sets what the receiving client's own player may do.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x76`  | Always `0x76`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Flags         | UByte      | `0b11`  | See below.                      |

| Bit | Name               | Meaning                                                                                                  |
|-----|--------------------|----------------------------------------------------------------------------------------------------------|
| 0   | `UNLIMITED_BLOCKS` | The player never runs out of blocks, and a [Block Line](../protocol075.md#block-line) may be any length. |
| 1   | `SPECTATOR_BUILD`  | A spectator may build and destroy blocks, from the camera position.                                      |
| 2   | `UNLIMITED_REACH`  | The player may build and destroy at any distance they can aim at.                                        |
| 3   | `NO_DELAY`         | No delay between two builds or two spade hits.                                                           |
| 4-7 | reserved           | Must be `0`. Clients ignore unknown bits.                                                                |

The flags apply until the next Edit Mode, and `0` restores the normal limits.
The map bounds and the water level still apply.

A spectator builds with [Block Action](../protocol075.md#block-action) and
[Block Line](../protocol075.md#block-line) under its own player id, in the colour
of its last [Set Colour](../protocol075.md#set-colour). Every client applies
them like any other player's.

The server still checks every block, and drops those it does not allow.

See [Extensions](extension.md) for how the extension is negotiated.
