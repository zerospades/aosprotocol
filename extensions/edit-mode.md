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
* may build and destroy at z `62`, on the water;
* has no delay between two builds or two spade hits;
* may build and destroy as a spectator.

The client offers these to the player, and the server may accept them. z `63`
never changes, and the server drops any block it does not allow.

The client may also let the player leave their body and fly the camera freely,
and return to it at any time. This is client side only. While flying, the player
only builds and destroys, from the camera, and the client keeps sending the
body's position and orientation.

A spectator sends [Block Action](../protocol075.md#block-action) `0`, `1` and
`2`, and [Block Line](../protocol075.md#block-line), under its own player id. Its
blocks take its colour from [Existing Player](../protocol075.md#existing-player),
then from each [Set Colour](../protocol075.md#set-colour) it sends, which the
server relays.

## Applying blocks

A client that negotiated Edit Mode applies every Block Action and Block Line it
receives this way, whoever sent it:

* a Block Line is drawn in full, whatever its length;
* a destroy, `1`, `2` or `3`, removes the blocks it covers at z `62` too, never
  at z `63`;
* a block stays while solid blocks touching by faces connect it to z `63`, and
  the others fall.

While any player is in edit mode, a client that did not negotiate Edit Mode does
not play. The server disconnects it, or makes it a spectator and sends it no
Block Action or Block Line, and may tell it in chat to update. Its map no longer
matches, so it stays out until it joins again or the next
[Map Start](../protocol075.md#map-start-075), even if edit mode ends first.

See [Extensions](extension.md) for how the extension is negotiated.
