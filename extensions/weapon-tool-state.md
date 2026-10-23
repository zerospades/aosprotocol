# Weapon and Tool State

Keeps the counts of the tool a player holds in step between client and server:
the gun's rounds, the blocks or the grenades. Today each side counts on its own:
the server from how long fire is held, the client from the rounds it fires, and
a reload the server refuses silently is finished by the client anyway. Here the
client predicts and the server corrects.

| ------------: | ------------- |
| Extension ID: | `0x39`        |
| Packet ID:    | `0x79`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name  | Direction         | Size |
|--------|-------|-------------------|------|
| 0      | State | Client <-> Server | 8    |

## Sub ID 0: State

The counts of the tool a player holds. From a client it is a claim, which lets
the client act at once; from the server it is the truth, which replaces the
client's.

| Field Name    | Field Type | Example | Notes                                          |
|---------------|------------|---------|------------------------------------------------|
| Packet ID     | UByte      | `0x79`  | Always `0x79`.                                 |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.                |
| Player ID     | UByte      | `7`     | Ignored from a client; the server fills it in. |
| Sequence      | UByte      | `41`    | See [Sequence](#sequence).                     |
| Tool          | UByte      | `2`     | The held tool, as in [Set Tool](../protocol075.md#set-tool). |
| Reloading     | UByte      | `0`     | Gun only: `1` while it reloads, else `0`. Other values read as `0`. |
| Count         | UByte      | `10`    | Rounds in the gun's magazine, blocks, or grenades. |
| Reserve       | UByte      | `50`    | Gun only: rounds carried beyond the magazine.  |

Only the held tool is described; which tool that is, and how a player switches,
is the business of [Set Tool](../protocol075.md#set-tool) or whatever extension
chooses it. A State whose Tool is not the one the receiver
thinks is held is ignored: it was sent across a switch.

### The client predicts

It counts on its own clock: when its reload ends it fills the magazine and may
fire at once. It claims only the gun: it sends a State when its reload ends,
when primary fire is released, and after each round of the shotgun's reload.
Blocks and grenades it does not claim, since the server counts every Block
Action and Grenade Packet exactly; their State comes from the server alone. It still asks for a reload with
[Weapon Reload](../protocol075.md#weapon-reload).

### The server corrects

It keeps its own counts and never takes the client's. It answers a claim when
they differ by more than it tolerates; how much that is, and how long after a
reload Reloading may still differ, is the server's choice. It also sends its
own State when the player switches to a gun, the block or a grenade, never to
a tool with nothing to count, and when it changes the counts
itself: when a reload starts, ends or is cancelled, a Restock, a pickup, a
respawn, and to a joining client.

To a client with the extension the server answers a Weapon Reload with a State,
not a Weapon Reload. To other clients with the extension it sends its own State
when a player's reload starts and ends, for their animation, and never relays a
client's claims; they may ignore the counts. Clients without the extension get
Weapon Reload as today. A server with the extension sends these counts to a
client that has it in State only, not in [Player Properties](player-properties.md).

### Sequence

A respawn, any switch of the held tool, even between two guns of a
[Loadout](loadout.md), or a new Loadout sets the Sequence to `0`. After it the
client ignores server States until the one the server sends for the switch
arrives, so a State about the gun it left is never applied to the new one. The client's first claim
after it carries `1`, and later claims count on from `2`, wrapping after `255`
back to `2`. After a reset the server ignores claims until one with `1` arrives,
so a claim still in transit from before is not taken for a new one.

The server's State carries the Sequence of the last claim it received, or `0`
when there has been none since the reset. It describes the counts as they were
at that claim, or at the reset for `0`, with only the server's own changes since
applied. What was used after the claim is left out. The client adopts it and
takes off what it has used since that claim, so nothing is counted twice.

### Reloads

Switching away from a gun cancels its reload on both sides. Only OpenSpades
does so today: other clients and the servers let the reload run on, so this is
new behaviour for them.

## Connection state

The counts belong to the player and last until
[Player Left](../protocol075.md#player-left).
[Map Start](../protocol075.md#map-start-075) does not clear them.

See [Extensions](extension.md) for how the extension is negotiated.
