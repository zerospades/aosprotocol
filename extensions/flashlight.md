# Flashlight

A light a player carries, switched by the server and drawn by every client. The
vanilla client has no flashlight.

| ------------: | ------------- |
| Extension ID: | `0x32`        |
| Packet ID:    | `0x72`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name         | Direction         | Size |
|--------|--------------|-------------------|------|
| 0      | Light        | Client <-> Server | 4    |
| 1      | Light State  | Server -> Client  | 2+   |
| 2      | Light Config | Server -> Client  | 8    |

A client only sends Light. The server drops any other sub-packet it receives
from a client.

## Sub ID 0: Light

Turns one player's light on or off.

| Field Name    | Field Type | Example | Notes                                         |
|---------------|------------|---------|-----------------------------------------------|
| Packet ID     | UByte      | `0x72`  | Always `0x72`.                                |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.               |
| Player ID     | UByte      | `0`     | The player whose light this is.               |
| State         | UByte      | `1`     | `0` off, `1` on. Any other value is on.       |

A client sends it to ask for its own light; the server ignores Player ID and
uses the sender's. The server decides whether to relay it, and a refused request
is simply not relayed. The server may also send it on its own.

## Sub ID 1: Light State

Every player's light, sent to a joining client after
[State Data](../protocol075.md#state-data).

| Field Name    | Field Type | Example  | Notes                                     |
|---------------|------------|----------|-------------------------------------------|
| Packet ID     | UByte      | `0x72`   | Always `0x72`.                            |
| Sub Packet ID | UByte      | `1`      | Always `1` for this sub-packet.           |
| States        | UByte[]    | `0b1010` | One bit per player id, rest of the packet. |

Bit `n` of byte `n / 8` is player id `n`, low bit first. An id past the end of
the array is off.

## Sub ID 2: Light Config

Sets one player's beam. The server sends it whenever it likes, for example when a
player equips a different flashlight, and to a joining client for every player
it has configured. Player ID `255` sets the beam of every player without a Light
Config of their own, including those who join later.

| Field Name    | Field Type | Example | Notes                                  |
|---------------|------------|---------|----------------------------------------|
| Packet ID     | UByte      | `0x72`  | Always `0x72`.                         |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet.        |
| Player ID     | UByte      | `0`     | `255` for every unconfigured player.   |
| Reach         | UByte      | `60`    | Blocks at which the light reaches zero. |
| Cone          | UByte      | `90`    | Full angle of the cone, in degrees.    |
| Red           | UByte      | `255`   | Linear, `255` is `1.0`.                |
| Green         | UByte      | `179`   |                                        |
| Blue          | UByte      | `128`   |                                        |

## Rendering

The light is a spotlight at the player's eye, pointing along the orientation the
client renders that player's model with.

A client draws every lit player's light, whichever team they are on, and only
those: a client that negotiated this extension does not draw a flashlight the
server has not switched on, its own included. It may approximate the light (a
drawn cone, a glow where it lands) but does not change its reach, cone or colour.

## Per-player state

A client turns off a player's light on
[Create Player](../protocol075.md#create-player),
[Kill Action](../protocol075.md#kill-action) and
[Player Left](../protocol075.md#player-left) for that player, and every light on
[Map Start](../protocol075.md#map-start-075). A player's Light Config
lasts until Player Left, surviving death, respawn and Map Start; the server may
send a new one at any time, for example on a map change. Light Config `255`
lasts for the connection. The server applies the same rules and sends nothing
for them.

See [Extensions](extension.md) for how the extension is negotiated.
