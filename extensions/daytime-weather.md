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
| Time          | UShort     | `0`     | `0` is night; anything else is day.   |
| Reserved      | UShort     | `0`     | Must be `0`.                          |
| Weather       | UByte[2]   | `0`     | Unimplemented in v1. Must be `0`.     |

The Reserved field is kept for a later version in which the time of day passes:
Time would then be the minutes since midnight and this field the speed at which
it advances. A version 1 client ignores its value.

## Drawing the time

By day, the client draws the world as it does without this extension.

By night, the sun casts no light and no shadow. The rest of the world's lighting
is scaled by `0.1`, and the fog and sky are drawn as the
[Fog Colour](../protocol075.md#fog-colour) times `0.1`. Dynamic lights, such as
flashlights, are not scaled.

A client that negotiated this extension waits for the first Sky before drawing
the world. The Time survives
[Map Start](../protocol075.md#map-start-075).

See [Extensions](extension.md) for how the extension is negotiated.
