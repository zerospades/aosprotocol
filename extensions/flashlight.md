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

A Cone above `179` is drawn as `179`. A Reach or Cone of `0` gives no light.

## Rendering

The light is a spotlight at the player's eye, pointing along the orientation the
client renders that player's model with.

A client draws every lit player's light, whichever team they are on, and only
those: a client that negotiated this extension does not draw a flashlight the
server has not switched on, its own included. It may approximate the light (a
drawn cone, a glow where it lands) but does not change its reach, cone or colour.

### The lamp

A client draws a glare in the beam's colour at the lamp of each lit player,
except the player it views the game through in first person. Distances are in
blocks and the z axis points down.

**Position.** The lamp is fixed to the head, which turns with the player's yaw and
pitches about a point 0.2 below the eye. Before pitching, the lamp is 0.36 ahead
of that point and 0.375 above it: with a level view, 0.36 ahead of the eye and
0.175 above it. Crouching does not move it relative to the eye.

**Strength.** With θ the angle between the beam's axis and the direction from the
lamp to the camera, and d the distance between them:

| Term     | Value                                                    |
|----------|----------------------------------------------------------|
| Beam     | `1 - smoothstep(min(θ / (Cone / 2), 1))`                  |
| Lens     | `cos θ`; nothing is drawn at `θ ≥ 90°`                    |
| Distance | `1 / (1 + (d / Reach)²)`                                  |
| Fog      | `1 - min(d_h² / 128², 1)`, `d_h` the horizontal distance  |

`smoothstep(x) = x²(3 - 2x)`. The glare follows Beam, leaving at most a faint
glint from Lens where Beam is `0`. Clients should scale it by Distance and Fog.

**Visibility.** A lamp inside a solid block shows nothing. Solid blocks on the
segment from the lamp to the camera hide the glare; players and models do not. A
client may ease it in and out over about 0.1 s.

Position, Beam, Lens and Visibility are required, so that every client gives a
lit player away alike. The look of the glare is free.

#### Reference rendering

The glare drawn over the finished frame, centred on the projected lamp, as two
additive layers of one radial profile, `r` from `0` at the centre to `1` at the
rim:

`alpha(r) = 1 - exp(-2 · (0.05 / (r + 0.05))² · (1 - r²)²)`

| Layer | Radius (screen heights) | Beam exponent | Glint | Colour             |
|-------|-------------------------|---------------|-------|--------------------|
| Glow  | 0.4                     | 1.5           | 0.05  | 20% towards white  |
| Core  | 0.12                    | 1             | 0.2   | white              |

Each layer's `amount` is `Beam^exponent + Glint · Lens`, times Distance, Fog,
visibility, the light's fade-in and the dark adaptation: `1` in daylight, up to
`8` in total darkness. The layer is drawn `Radius · √amount` wide and
`min(√amount, 1)` bright, so more light widens the glare instead of clipping it.

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
