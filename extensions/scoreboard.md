# Scoreboard

Player and team scores set by the server, how many teams the scoreboard shows,
their colours and names, and who is on it.

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
| 5      | Team Info       | Server -> Client | `7+name`      |
| 6      | Listed          | Server -> Client | 4             |

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
| Team ID       | UByte      | `0`     | `0`-`254`.                      |
| Score         | LE Int     | `3`     | Signed, absolute.               |

## Sub ID 3: Resync Request

Asks the server for a [Score Table](#sub-id-1-score-table). The server may
ignore it.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet. |

## Sub ID 4: Scoreboard Mode

How many teams the scoreboard shows. Sent with the
[Score Table](#sub-id-1-score-table) and whenever it changes. Until the first
one, the mode is `2`, the base protocol's scoreboard.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `4`     | Always `4` for this sub-packet. |
| Mode          | UByte      | `2`     | See below.                      |

| Mode  | Name     | Scoreboard                                                                                                       |
|-------|----------|------------------------------------------------------------------------------------------------------------------|
| 0     | `SOLO`   | Every player is a team of one: a single list ranked by score, no team score, and no player is anyone's teammate. |
| 1     | 1 team   | One column, team `0`, its players ranked by score, with the team score.                                          |
| 2     | 2 teams  | A column per team, teams `0` and `1`, as in the base protocol.                                                   |
| 3     | 3 teams  | A column per team, teams `0` to `2`.                                                                             |
| 4     | 4 teams  | A column per team, teams `0` to `3`.                                                                             |
| 5-255 | reserved | Treated as `2`.                                                                                                  |

In a mode of N teams, the team of a player in
[Existing Player](../protocol075.md#existing-player),
[Short Player Data](../protocol075.md#short-player-data),
[Create Player](../protocol075.md#create-player) and
[Change Team](../protocol075.md#change-team) is `0` to N-1, or `255` for a
spectator. Spectators are listed apart from the teams, in every mode.

## Sub ID 5: Team Info

Gives one team its colour and name. Any team id from `0` to `254` can be
defined, up to 255 teams, whether or not the current mode shows it. Sent with
the [Score Table](#sub-id-1-score-table) for every team past `1` and every
override of `0` or `1`, and whenever a colour or name changes.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `5`     | Always `5` for this sub-packet. |
| Team ID       | UByte      | `2`     | `0`-`254`.                      |
| Blue          | UByte      | `0`     |                                 |
| Green         | UByte      | `0`     |                                 |
| Red           | UByte      | `255`   |                                 |
| Name          | String     | `"Red"` | NUL-terminated, not empty.      |

The name is encoded as a [Chat Message](../protocol075.md#chat-message) is: Code
Page 437, or UTF-8 behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). It
is not limited to the 10 characters of
[State Data](../protocol075.md#state-data). A client drops a Team Info with an
empty name.

### Defined teams

Every team has a colour and a name. Teams `0` and `1` take theirs from State
Data, which a Team Info for them overrides. Any other team has neither until its
Team Info arrives, and until then it does not exist:

* the server puts no player in it, and sends no
  [Team Score](#sub-id-2-team-score) for it;
* a client does not offer it, and sends no
  [Change Team](../protocol075.md#change-team) to it; a server refuses one that
  does, and leaves the player where they were;
* a server sends a [Scoreboard Mode](#sub-id-4-scoreboard-mode) only once every
  team it shows is defined.

## Sub ID 6: Listed

Puts a player on the scoreboard or takes them off it. Until the first one, every
player is listed.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `6`     | Always `6` for this sub-packet. |
| Player ID     | UByte      | `0`     |                                 |
| Listed        | UByte      | `0`     | `0` off the scoreboard, `1` on. |

A player off the scoreboard is left out of it and of the player counts, and no
join, team change or leave notification is shown for them. Their kills and
deaths are still shown: a [Kill Action](../protocol075.md#kill-action) they make
or suffer reaches the kill feed as any other. They are drawn, heard and hit as
any other player.

To keep a joining player's arrival quiet, the server sends Listed before the
[Existing Player](../protocol075.md#existing-player) or
[Create Player](../protocol075.md#create-player) that introduces them. A client
always lists its own player.

## Scores

A client never changes a score itself, on
[Kill Action](../protocol075.md#kill-action),
[Intel Capture](../protocol075.md#intel-capture),
[Territory Capture](../protocol075.md#territory-capture) or any other event. The
server sends every change. Where [Player Properties](player-properties.md) also
carries a score, the later packet wins.

[Player Left](../protocol075.md#player-left) clears that player's score and
lists the id again, after the leave is handled, so an unlisted player leaves
silently. [Map Start](../protocol075.md#map-start-075) clears every player and
team score, undefines every team past `1`, restores teams `0` and `1` to the
State Data that follows it, and lists every player again.
[Create Player](../protocol075.md#create-player) keeps all of it.

See [Extensions](extension.md) for how the extension is negotiated.
