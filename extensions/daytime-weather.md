# Daytime and Weather

The time of day and the weather, set by the server.

| ------------: | ------------- |
| Extension ID: | `0x33`        |
| Packet ID:    | `0x73`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

## Sub ID 0: Sky

Server to client only. Sent to a joining client after
[State Data](../protocol075.md#state-data), and at any time after; it applies on
arrival.

| Field Name    | Field Type | Example | Notes                                 |
|---------------|------------|---------|---------------------------------------|
| Packet ID     | UByte      | `0x73`  | Always `0x73`.                        |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.       |
| Time          | UShort     | `0`     | Minutes since midnight, `0`-`1439`.   |
| Weather       | UByte[2]   | `0`     | Reserved. Must be `0`.                |

Version 1 draws `0` (12 AM) as complete darkness: black sky and fog, no sunlight.
It draws `720` (12 PM) as daylight, as without this extension. Other times are
drawn as the nearer of the two. A Time above `1439` is ignored.

Before the first Sky the client draws daylight. The Time survives
[Map Start](../protocol075.md#map-start-075).

See [Extensions](extension.md) for how the extension is negotiated.
