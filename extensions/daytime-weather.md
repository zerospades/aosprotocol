# Daytime and Weather

The time of day and the weather, set by the server.

| ------------: | ------------- |
| Extension ID: | `0x33`        |
| Packet ID:    | `0x73`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

## Sub ID 0: Sky

Server to client only. Sent as soon as the server knows the client negotiated
this extension, and at any time after; it applies on arrival.

| Field Name    | Field Type | Example | Notes                                 |
|---------------|------------|---------|---------------------------------------|
| Packet ID     | UByte      | `0x73`  | Always `0x73`.                        |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.       |
| Time          | UShort     | `0`     | Minutes since midnight, `0`-`1439`.   |
| Speed         | UShort     | `60`    | Game minutes per real minute.         |
| Weather       | UByte[2]   | `0`     | Unimplemented in v1. Must be `0`.     |

The client advances Time by Speed from the moment the Sky arrives, wrapping at
`1440`. `0` stops the clock and `1` is real time; a day lasts `1440 / Speed` real
minutes, so `60` gives a 24-minute day.

## Drawing the time

The client places the sun from Time alone; the server never sends it. In map axes
(`+x` east, `+y` south, `+z` down), the sun turns once a day around the axis
`(0, 1, -1)`, 45° up to the south. At `720` (12 PM) it is at `(0, -1, -1)`, 45°
up to the north. It rises in the east at `360` (6 AM) and sets in the west at
`1080` (6 PM). These are map axes, not [Teamplay's North](teamplay.md#north),
which does not move the sun.

With `a = (Time - 720) / 4`, the degrees the sun has turned since noon, the
daylight is `D = max(0.1, clamp(2 cos a, 0, 1))`. The world's lighting is scaled
by `D`, and the fog and sky are drawn as the
[Fog Colour](../protocol075.md#fog-colour) times `D`. So 6 PM to 6 AM is night at
`0.1`, 8 AM to 4 PM is full daylight, and `720` looks as it does without this
extension.

While the sun is below the horizon, from 6 PM to 6 AM, it casts no light and no
shadow; the night's light falls evenly on every face.

A client that negotiated this extension waits for the first Sky before drawing
the world. The Time survives
[Map Start](../protocol075.md#map-start-075).

See [Extensions](extension.md) for how the extension is negotiated.
