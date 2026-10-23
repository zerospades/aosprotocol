# Loadout

The server sets what a player carries in hand: which tools, in which order, and
how many of each. A loadout can be a spade and a block, a grenade
and an empty hand, or a rifle, a shotgun and a spade. Each tool behaves as its
base tool does.

| ------------: | ------------- |
| Extension ID: | `0x3B`        |
| Packet ID:    | `0x7B`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name    | Direction         | Size |
|--------|---------|-------------------|------|
| 0      | Loadout | Server -> Client  | 5+   |
| 1      | Hold    | Client <-> Server | 5    |

## Sub ID 0: Loadout

One player's slots. The server sends it to every client, for every player, so
that each can draw what others hold. A server receiving one drops it.

| Field Name    | Field Type | Example | Notes                                   |
|---------------|------------|---------|-----------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                          |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.         |
| Player ID     | UByte      | `7`     | Whose loadout this is.                  |
| Slot Count    | LE UShort  | `3`     | Slots that follow.                      |
| Slots         | Slot[]     |         | See [Slot](#slot).                      |

The server sends a player's Loadout once after their
[Existing Player](../protocol075.md#existing-player) or first
[Create Player](../protocol075.md#create-player), and then only when it changes
it, each time followed by a [Hold](#sub-id-1-hold). A new Loadout takes
effect at once and starts full.

### Slot

Always three bytes, whatever the Tool, so a client skips a slot it does not
know.

| Field Name | Field Type | Example | Notes                                        |
|------------|------------|---------|----------------------------------------------|
| Tool       | UByte      | `2`     | See below.                                   |
| Flags      | UByte      | `0b0`   | Bit `0` `INFINITE`: the slot's count is never used up. Bits `1`-`7` must be `0` and are ignored. |
| Value      | UByte      | `1`     | Depends on the Tool, see below.              |

| Tool    | Value                                                                  |
|---------|------------------------------------------------------------------------|
| Gun     | The weapon: `0` rifle, `1` SMG, `2` shotgun, as in [Change Weapon](../protocol075.md#change-weapon). |
| Block   | Blocks after a [Restock](../protocol075.md#restock).                    |
| Grenade | Grenades after a Restock.                                              |
| Others  | `0`, ignored.                                                          |

Each slot keeps its own count, so a loadout can carry three frag grenades and
two smoke canisters. With `INFINITE` the count is ignored and the client shows
none; on a gun it is the reserve that is never used up, so a reload is always
allowed.

| Value     | Tool       | Primary fire                               |
|-----------|------------|--------------------------------------------|
| `0`       | Spade      | As in the base game.                       |
| `1`       | Block      | As in the base game.                       |
| `2`       | Gun        | As the base weapon in Weapon.              |
| `3`       | Grenade    | As in the base game.                       |
| `4`       | Empty hand | Nothing.                                   |
| `5`-`255` | reserved   | A client treats the slot as an empty hand. |

Slots are numbered from `0` in the order the player cycles through them. A
Loadout says what each slot does; [Cosmetic Weapon and
Tool](cosmetic-weapon-tool.md) can give each one its look, by the same numbers,
and is sent after the Loadout. Each gun slot keeps its own magazine and reserve,
and [Weapon and Tool State](weapon-tool-state.md) counts the one held.

## Sub ID 1: Hold

Which slot a player holds. A client with a loadout sends Hold rather than
[Set Tool](../protocol075.md#set-tool) when its player switches.

| Field Name    | Field Type | Example | Notes                                          |
|---------------|------------|---------|------------------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                                 |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.                |
| Player ID     | UByte      | `7`     | Ignored from a client; the server fills it in. |
| Slot          | LE UShort  | `2`     | An index into the player's Slots.              |

The server relays it to clients with the extension, and sends the others a base
Set Tool for the slot's Tool, with an empty hand as the spade. It also sends the
Hold it settles on back to the sender: a Hold sent just before a new Loadout
arrived can name a slot that still exists but now holds something else, and the
server's Hold corrects it. A Slot beyond the player's Slot Count is ignored.

## Base packets

* **Create Player** and **Existing Player** carry, for clients without the
  extension, the weapon of the player's first gun slot, or the rifle when there
  is none, and base packets carry the held slot's counts. A respawn refills the
  loadout, magazines included.
* **Set Tool** from a client is handled as in the base game until its Loadout
  arrives. Once a loadout is set, the server reads it as a Hold for the first
  slot of that Tool, and ignores it when no slot has that Tool.

## Connection state

A player's loadout lasts until another replaces it or
[Player Left](../protocol075.md#player-left).
[Map Start](../protocol075.md#map-start-075) does not clear it.

See [Extensions](extension.md) for how the extension is negotiated.
