# Scoreboard

Server-authoritative scores. Without it a client has to derive score itself —
`+1` to the killer on every [Kill Action](../protocol075.md#kill-action) — which
is wrong on any server with non-unit scoring, teamkill penalties or objective
points, and stays wrong for the rest of the round.

| ------------: | ------------- |
| Extension ID: | 5             |
| Packet ID:    | 69            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

The packet id is `64 + extension id`, see
[Extension IDs](extension.md#extension-ids).

### Sub Packets:

| Sub ID | Name            | Direction        | Size |
|--------|-----------------|------------------|------|
| 0      | Score Update    | Server -> Client | 7    |
| 1      | Score Table     | Server -> Client | 2+   |
| 2      | Team Score      | Server -> Client | 7    |
| 3      | Resync Request  | Client -> Server | 2    |

Sub ids 0-2 are server to client; a server that receives one from a client drops
it. Sub id 3 is the only packet a client sends.

## Sub ID 0: Score Update

One player's score. Sent whenever it changes, for any reason.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `69`    | Always `69`.                    |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Player ID     | UByte      | `0`     |                                 |
| Score         | LE Int     | `7`     | Signed, absolute.               |

Absolute and not a delta: a delta is only correct if the receiver's base is,
which is the failure this extension exists to fix. Applying the same Score Update
twice leaves the same score.

Signed because a teamkill penalty is one of the cases that breaks the derived
score. This is also the field
[Player Properties](player-properties.md) leaves ambiguous; here it is `int32`.

## Sub ID 1: Score Table

Every score at once, for a client that has just joined. The server sends it after
[State Data](../protocol075.md#state-data), and in response to a
[Resync Request](#sub-id-3-resync-request).

| Field Name    | Field Type | Example | Notes                                     |
|---------------|------------|---------|-------------------------------------------|
| Packet ID     | UByte      | `69`    | Always `69`.                              |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.           |
| Entries       | Entry[]    |         | The remaining bytes, 5 each. See below.   |

**Entry**

| Field Name | Field Type | Example | Notes             |
|------------|------------|---------|-------------------|
| Player ID  | UByte      | `0`     |                   |
| Score      | LE Int     | `7`     | Signed, absolute. |

The entry count is implied by the packet length. Sparse rather than an array
indexed by player id: 5 bytes per connected player beats 1024 bytes of mostly
absent ids under [Player Limit](player-limit.md). An id absent from the table
scores `0`. A trailing partial entry makes the packet malformed and the receiver
drops it whole.

## Sub ID 2: Team Score

| Field Name    | Field Type | Example | Notes                                    |
|---------------|------------|---------|------------------------------------------|
| Packet ID     | UByte      | `69`    | Always `69`.                             |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet.          |
| Team ID       | UByte      | `0`     | `0` and `1` as in the base protocol.     |
| Score         | LE Int     | `3`     | Signed, absolute.                        |

[CTF State](../protocol075.md#ctf-state) carries a team score as a `UByte`, so it
saturates at `255` and exists only in CTF. This one does not and is sent in any
mode. Where both are present this is authoritative.

## Sub ID 3: Resync Request

Asks the server for a [Score Table](#sub-id-1-score-table). No payload.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `69`    | Always `69`.                    |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet. |

A request, like everything a client sends: the server may ignore it, and should
rate limit it. A client sends one when it has reason to believe its table is
stale, not on a timer and not to poll.

## The client does not derive score

While this extension is negotiated:

* A client **must not** change any score on
  [Kill Action](../protocol075.md#kill-action). The server sends a Score Update
  for the kill it just announced.
* A client **must not** change any score on
  [Intel Capture](../protocol075.md#intel-capture),
  [Territory Capture](../protocol075.md#territory-capture) or any other event.
* A server **must** send a Score Update for every change, including changes that
  a client could have inferred.

Scores the client cannot see are the point; a client that keeps guessing
alongside the server gets two answers and no rule for choosing.

## Interaction with Player Properties

[Player Properties](player-properties.md) also carries a Score field. Both are
the server speaking and both are reliable and ordered, so the later packet wins.
A server that negotiates both keeps them consistent.

Ammunition, blocks and grenades stay in Player Properties. Restating them here
would make a second source of truth for state that already has one.

## Per-player state

* [Player Left](../protocol075.md#player-left) clears the score for that id. Ids
  are recycled.
* [Create Player](../protocol075.md#create-player) does not. A respawn, a team
  change and a weapon change all keep the score; whether a team change should
  reset it is the server's decision, sent as a Score Update.
* [Map Start](../protocol075.md#map-start-075) clears every score, player and
  team, to `0`. The server sends nothing for it.

## Version growth

Version 1 defines sub ids `0`-`3`. A receiver drops a sub id it does not know,
so a version 2 may add sub-packets without breaking version 1 — but it may not
change the layout of `0`-`3`, which is what makes that safe.

See [Extensions](extension.md) for how the extension is negotiated.
