# Extended Spawn Packet

Per-player properties the base protocol has no room for: flags that leave a
player out of what other clients report about players, and a colour that
replaces their team colour.

| ------------: | ------------- |
| Extension ID: | `0x34`        |
| Packet ID:    | `0x74`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name                     | Direction        | Size  |
|--------|--------------------------|------------------|-------|
| 0      | Extended Create Player   | Server -> Client | `21+` |
| 1      | Extended Existing Player | Server -> Client | `17+` |
| 2      | Extended State Data      | Server -> Client | `36+` |

To a client that negotiated this extension, the server sends sub 0 instead of
[Create Player](../protocol075.md#create-player), sub 1 instead of
[Existing Player](../protocol075.md#existing-player) and sub 2 instead of
[State Data](../protocol075.md#state-data). Other extensions treat
them as the packets they replace.

## Flags

| Bit | Name              | Meaning                                                                     |
|-----|-------------------|-----------------------------------------------------------------------------|
| 0   | `HIDE_SCOREBOARD` | Left out of the scoreboard and its counts, and of spectator camera cycling. |
| 1   | `HIDE_PRESENCE`   | No join, team change or leave notification.                                 |
| 2   | `HIDE_KILLFEED`   | No kill feed line for a kill this player made or suffered.                  |
| 3   | `NO_STATS`        | Ignored by client-side statistics such as kill counters and streaks.        |
| 4   | `CUSTOM_COLOR`    | Drawn in the player's [colour](#colour) instead of the team colour.         |
| 5   | `HIDE_MAP`        | Left off the minimap and the map view.                                      |
| 6-7 | reserved          | Must be `0`. Clients ignore unknown bits.                                   |

Bits 0 to 3 and 5 change what is reported, not what is drawn: the player is still
rendered, heard and hit as usual. A client ignores them for its own player, and
still tells its player about their own kills and deaths.

## Colour

Blue, green, red, as in [Set Colour](../protocol075.md#set-colour). Used only
while `CUSTOM_COLOR` is set. It replaces the team colour on the player model,
the tool or weapon they hold, and their corpse. It does not change their team.

## Team

`0` and `1` as in the base protocol, `255` spectator. `2` to `254` are further
teams, listed by [Extended State Data](#sub-id-2-extended-state-data). Their
players are drawn and hit like any other player, and are teammates only of their
own team: for the minimap, name tags, the spectator camera and friendly fire.
Further teams do not appear in the scoreboard. A team the client has no entry
for has an empty name and the colour `128, 128, 128`.

## Sub ID 0: Extended Create Player

| Field Name    | Field Type   | Example  | Notes                                          |
|---------------|--------------|----------|------------------------------------------------|
| Packet ID     | UByte        | `0x74`   | Always `0x74`.                                 |
| Sub Packet ID | UByte        | `0`      | Always `0` for this sub-packet.                |
| Player ID     | UByte        | `254`    |                                                |
| Flags         | UByte        | `0b1011` | See [Flags](#flags).                           |
| Weapon        | UByte        | `0`      | As in Create Player.                           |
| Team          | UByte        | `0`      | See [Team](#team).                             |
| X position    | LE Float     | `256.0`  | As in Create Player.                           |
| Y position    | LE Float     | `256.0`  | As in Create Player.                           |
| Z position    | LE Float     | `40.0`   | As in Create Player.                           |
| Colour        | UByte[3]     |          | See [Colour](#colour).                         |
| Name          | CP437 String | `Wolf`   | As in Create Player, to the end of the packet. |

## Sub ID 1: Extended Existing Player

| Field Name    | Field Type   | Example  | Notes                                            |
|---------------|--------------|----------|--------------------------------------------------|
| Packet ID     | UByte        | `0x74`   | Always `0x74`.                                   |
| Sub Packet ID | UByte        | `1`      | Always `1` for this sub-packet.                  |
| Player ID     | UByte        | `254`    |                                                  |
| Flags         | UByte        | `0b1011` | See [Flags](#flags).                             |
| Team          | UByte        | `0`      | See [Team](#team).                               |
| Weapon        | UByte        | `0`      | As in Existing Player.                           |
| Held item     | UByte        | `0`      | As in Existing Player.                           |
| Kills         | LE UInt      | `0`      | As in Existing Player.                           |
| Block Colour  | UByte[3]     |          | Blue, green, red, as in Existing Player.         |
| Colour        | UByte[3]     |          | See [Colour](#colour).                           |
| Name          | CP437 String | `Wolf`   | As in Existing Player, to the end of the packet. |

## Sub ID 2: Extended State Data

| Field Name    | Field Type  | Example | Notes                                    |
|---------------|-------------|---------|------------------------------------------|
| Packet ID     | UByte       | `0x74`  | Always `0x74`.                           |
| Sub Packet ID | UByte       | `2`     | Always `2` for this sub-packet.          |
| Player ID     | UByte       | `0`     | As in State Data.                        |
| Fog Colour    | UByte[3]    |         | Blue, green, red, as in State Data.      |
| Team Count    | UByte       | `3`     |                                          |
| Teams         | TeamEntry[] |         | Team Count entries.                      |
| Game Mode     | UByte       | `0`     | As in State Data.                        |
| Mode State    |             |         | CTF State or TC State, as in State Data. |

**TeamEntry** (14 bytes)

| Field Name | Field Type   | Example  | Notes                      |
|------------|--------------|----------|----------------------------|
| Team ID    | UByte        | `2`      | See [Team](#team).         |
| Colour     | UByte[3]     |          | Blue, green, red.          |
| Name       | CP437 String | `Zombie` | Always 10 characters long. |

Teams `0` and `1` are always listed, and `255` never. Ids need not follow each
other.

## Lifetime

The flags and colour belong to the player id, and each sub-packet replaces
them. [Player Left](../protocol075.md#player-left) resets them to
`0` once it is applied, so a silent player leaves silently.
[Map Start](../protocol075.md#map-start-075) resets every id.

## Notes

Ids above `31` need [Player Limit](player-limit.md), so servers allocate silent
ids downwards from `254`.

See [Extensions](extension.md) for how the extension is negotiated.
