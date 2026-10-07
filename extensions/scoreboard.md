# Scoreboard

Player and team scores set by the server, and how the scoreboard groups players.

| ------------: | ------------- |
| Extension ID: | `0x37`        |
| Packet ID:    | `0x77`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name            | Direction        | Size          |
|--------|-----------------|------------------|---------------|
| 0      | Score Update    | Server -> Client | 7             |
| 1      | Score Table     | Server -> Client | `2+5*entries` |
| 2      | Team Score      | Server -> Client | 7             |
| 3      | Resync Request  | Client -> Server | 2             |
| 4      | Scoreboard Mode | Server -> Client | 3             |

Sub id 3 is the only packet a client sends; a server drops any other it receives
from a client.

## Sub ID 0: Score Update

One player's score, sent whenever it changes.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Player ID     | UByte      | `0`     |                                 |
| Score         | LE Int     | `7`     | Signed, absolute.               |

## Sub ID 1: Score Table

Every player's score. Sent after [State Data](../protocol075.md#state-data) and
in answer to a [Resync Request](#sub-id-3-resync-request).

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet. |
| Entries       | Entry[]    |         | To the end of the packet.       |

**Entry** (5 bytes)

| Field Name | Field Type | Example | Notes             |
|------------|------------|---------|-------------------|
| Player ID  | UByte      | `0`     |                   |
| Score      | LE Int     | `7`     | Signed, absolute. |

A player the table does not list scores `0`.

## Sub ID 2: Team Score

One team's score, sent whenever it changes, in any game mode. It takes
precedence over the score in [CTF State](../protocol075.md#ctf-state).

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet. |
| Team ID       | UByte      | `0`     | Any team the client knows.      |
| Score         | LE Int     | `3`     | Signed, absolute.               |

## Sub ID 3: Resync Request

Asks the server for a [Score Table](#sub-id-1-score-table). The server may
ignore it.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet. |

## Sub ID 4: Scoreboard Mode

How the scoreboard groups players. Sent with the
[Score Table](#sub-id-1-score-table) and whenever the mode changes. Until the
first one, the mode is `0`.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `4`     | Always `4` for this sub-packet. |
| Mode          | UByte      | `1`     | See below.                      |

| Mode  | Name     | Scoreboard                                                                                                       |
|-------|----------|------------------------------------------------------------------------------------------------------------------|
| 0     | `TEAMS`  | A column per team, its players ranked by score, with the team score.                                             |
| 1     | `SOLO`   | Every player is a team of one: a single list ranked by score, no team score, and no player is anyone's teammate. |
| 2-255 | reserved | Treated as `0`.                                                                                                  |

## Scores

A client never changes a score itself, on
[Kill Action](../protocol075.md#kill-action),
[Intel Capture](../protocol075.md#intel-capture),
[Territory Capture](../protocol075.md#territory-capture) or any other event. The
server sends every change. Where [Player Properties](player-properties.md) also
carries a score, the later packet wins.

[Player Left](../protocol075.md#player-left) clears that player's score.
[Map Start](../protocol075.md#map-start-075) clears every player and team score.
[Create Player](../protocol075.md#create-player) keeps it.

See [Extensions](extension.md) for how the extension is negotiated.
