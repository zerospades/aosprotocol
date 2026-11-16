# Scoreboard

Player and team scores set by the server, the score that wins, how many teams
the scoreboard shows, their colours and names, and who is on it.

| ------------: | ------------- |
| Extension ID: | `0x37`        |
| Packet ID:    | `0x77`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name            | Direction        | Size                     |
|--------|-----------------|------------------|--------------------------|
| 0      | Score Update    | Server -> Client | 7                        |
| 1      | Score Table     | Server -> Client | `3+5*teams+5*players`    |
| 2      | Team Score      | Server -> Client | 7                        |
| 3      | Scoreboard Mode | Server -> Client | 3                        |
| 4      | Team Info       | Server -> Client | `7+name`                 |
| 5      | Listed          | Server -> Client | 4                        |
| 6      | Target Score    | Server -> Client | 6                        |

Every sub-packet is sent by the server. A server drops one it receives from a
client.

## Sequence

A client joining, or reloading after [Map Start](../protocol075.md#map-start-075),
receives, in this order:

1. after the map, before the first
   [Existing Player](../protocol075.md#existing-player): a
   [Team Info](#sub-id-4-team-info) for every defined team past `1` and every
   override of `0` or `1`, then the [Scoreboard Mode](#sub-id-3-scoreboard-mode),
   the [Target Score](#sub-id-6-target-score), and a [Listed](#sub-id-5-listed)
   for every player off the scoreboard;
2. the base protocol's Existing Player for every player, then
   [State Data](../protocol075.md#state-data);
3. after State Data: the [Score Table](#sub-id-1-score-table).

A packet of step 1 the server has nothing to say with is left out: a mode of
`2`, no target, no team past `1`, every player listed. During the game each
packet is sent when what it carries changes, and takes effect at once.

## Sub ID 0: Score Update

One player's score.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Player ID     | UByte      | `0`     |                                 |
| Score         | LE Int     | `7`     | Signed, absolute.               |

## Sub ID 1: Score Table

Every team's and every player's score, sent in step 3 of the
[Sequence](#sequence).

| Field Name    | Field Type   | Example | Notes                           |
|---------------|--------------|---------|---------------------------------|
| Packet ID     | UByte        | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte        | `1`     | Always `1` for this sub-packet. |
| Team Count    | UByte        | `2`     | Team entries that follow.       |
| Teams         | Team Entry[] |         |                                 |
| Players       | Entry[]      |         | To the end of the packet.       |

**Team Entry** and **Entry** (5 bytes each)

| Field Name        | Field Type | Example | Notes             |
|-------------------|------------|---------|-------------------|
| Team or Player ID | UByte      | `0`     |                   |
| Score             | LE Int     | `7`     | Signed, absolute. |

A team or player the table does not list scores `0`. A client drops a table
whose player entries do not fill the rest of the packet exactly.

## Sub ID 2: Team Score

One team's score, in any game mode. It takes precedence over the scores in
[CTF State](../protocol075.md#ctf-state).

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet. |
| Team ID       | UByte      | `0`     | A [defined team](#defined-teams). |
| Score         | LE Int     | `3`     | Signed, absolute.               |

## Sub ID 3: Scoreboard Mode

How many teams the game has. Until the first one, the mode is `2`, the base
protocol's.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet. |
| Mode          | UByte      | `2`     | See below.                      |

| Mode  | Name     | Scoreboard                                                     |
|-------|----------|----------------------------------------------------------------|
| 0     | `SOLO`   | A single list ranked by score, no team score.                  |
| 1     | 1 team   | One column, team `0`, ranked by score, with the team score.    |
| 2     | 2 teams  | A column per team, teams `0` and `1`, as in the base protocol. |
| 3     | 3 teams  | A column per team, teams `0` to `2`.                           |
| 4     | 4 teams  | A column per team, teams `0` to `3`.                           |
| 5-255 | reserved | Never sent. A client ignores the packet and keeps its mode.    |

A new mode comes with a new version of the extension.

In a mode of N teams, a player's team in
[Existing Player](../protocol075.md#existing-player),
[Short Player Data](../protocol075.md#short-player-data),
[Create Player](../protocol075.md#create-player) and
[Change Team](../protocol075.md#change-team) is `0` to N-1, or `255` for a
spectator. No player is ever in a team the mode does not have. Spectators are
listed apart, in every mode.

In `SOLO` every player is on team `0`, which gives them its colour and nothing
else. A client treats every other player as an enemy, whatever the team field
says: no name tag or map marker as a teammate, and its hits are reported as on
any enemy. Team chat is sent as usual; whom it reaches is the server's choice.

To change the mode, the server keeps every player in a team that exists:

* to add teams: [Team Info](#sub-id-4-team-info) for each new team, then the
  Scoreboard Mode, then any player moved into them;
* to remove teams: every player moved out of them, then the Scoreboard Mode.

## Sub ID 4: Team Info

Gives one team its colour and name.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `4`     | Always `4` for this sub-packet. |
| Team ID       | UByte      | `2`     | `0`-`254`.                      |
| Blue          | UByte      | `0`     |                                 |
| Green         | UByte      | `0`     |                                 |
| Red           | UByte      | `255`   |                                 |
| Name          | String     | `"Red"` | NUL-terminated, not empty.      |

The name is encoded as a [Chat Message](../protocol075.md#chat-message) is: Code
Page 437, or UTF-8 behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). It
is not limited to the 10 characters of
[State Data](../protocol075.md#state-data). A client drops a Team Info with an
empty name or a Team ID of `255`.

### Defined teams

Every team has a colour and a name. Teams `0` and `1` take theirs from State
Data. A Team Info for either overrides it until the next
[Map Start](../protocol075.md#map-start-075), whichever of the two arrives
first. Any other team has neither until its Team Info arrives, and until then it
does not exist: no player is put in it, no score is sent for it, a client does
not offer it, and a server refuses a [Change Team](../protocol075.md#change-team)
to it, leaving the player where they were.

Teams `0` to `254` can be defined, up to 255. A defined team past the mode's
last holds no player; it is kept for a later mode.

## Sub ID 5: Listed

Puts a player on the scoreboard or takes them off it. Until the first one, every
player is listed.

| Field Name    | Field Type | Example | Notes                                      |
|---------------|------------|---------|--------------------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                             |
| Sub Packet ID | UByte      | `5`     | Always `5` for this sub-packet.            |
| Player ID     | UByte      | `0`     |                                            |
| Listed        | UByte      | `0`     | `0` off the scoreboard. Any other value on. |

A player off the scoreboard is left out of it and of the player counts, and no
join, team change or leave notification is shown for them. Their kills and
deaths are still shown: a [Kill Action](../protocol075.md#kill-action) they make
or suffer reaches the kill feed as any other. They are drawn, heard and hit as
any other player, and their scores are still sent.

For a player joining during the game, the server sends Listed before the
[Existing Player](../protocol075.md#existing-player) or
[Create Player](../protocol075.md#create-player) that introduces them, so the
join is quiet. A client always lists its own player.

Where the Extended Spawn Packet extension is also negotiated, a
player is off the scoreboard if either extension takes them off it.

## Sub ID 6: Target Score

The score that wins: a team's, or in `SOLO` a player's. One target applies to
all of them. It takes precedence over the capture limit in
[CTF State](../protocol075.md#ctf-state); until the first one, that capture
limit applies in CTF and there is no target in TC.

| Field Name    | Field Type | Example | Notes                              |
|---------------|------------|---------|------------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                     |
| Sub Packet ID | UByte      | `6`     | Always `6` for this sub-packet.    |
| Target        | LE Int     | `10`    | Signed. `0` or less: no target.    |

A team or player reaches the target at a score at or above it. A client shows
it and never ends the game on it; the server decides the win.

## Scores

A client never changes a score itself, on
[Kill Action](../protocol075.md#kill-action),
[Intel Capture](../protocol075.md#intel-capture),
[Territory Capture](../protocol075.md#territory-capture) or any other event. The
server sends every change. The score in
[Existing Player](../protocol075.md#existing-player) and in
[Player Properties](player-properties.md) is ignored: this extension is the only
source of scores.

## Reset

[Player Left](../protocol075.md#player-left) clears that player's score and
lists the id again, after the leave is handled, so an unlisted player leaves
silently.

[Map Start](../protocol075.md#map-start-075) clears every player and team score
and the target, undefines every team past `1`, drops the overrides of `0` and
`1`, sets the mode to `2` and lists every player again. The
[Sequence](#sequence) that follows restores what the server still wants.

[Create Player](../protocol075.md#create-player) changes none of it: scores, team
info, mode, target and listing all survive a respawn.

See [Extensions](extension.md) for how the extension is negotiated.
