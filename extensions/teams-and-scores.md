# Teams and Scores

How many teams a map is played with, their colours, names and shown events,
player and team scores set by the server, the score that wins, and who is on the
scoreboard.

| ------------: | ------------- |
| Extension ID: | `0x37`        |
| Packet ID:    | `0x77`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name         | Direction        | Size     |
|--------|--------------|------------------|----------|
| 0      | Score        | Server -> Client | 7        |
| 1      | Team Score   | Server -> Client | 7        |
| 2      | Team Count   | Server -> Client | 3        |
| 3      | Team Info    | Server -> Client | `8+name` |
| 4      | Listed       | Server -> Client | 4        |
| 5      | Target Score | Server -> Client | 6        |

## Sequence

A client joining, or reloading after [Map Start](../protocol075.md#map-start-075),
receives, in this order:

1. before the first [Existing Player](../protocol075.md#existing-player): the
   [Team Count](#sub-id-2-team-count), a [Team Info](#sub-id-3-team-info) for
   every team past `1` and every override of `0` or `1`, and a
   [Listed](#sub-id-4-listed) for every player off the scoreboard;
2. the base protocol's Existing Player for every player, then
   [State Data](../protocol075.md#state-data);
3. after State Data: the [Target Score](#sub-id-5-target-score), and a
   [Score](#sub-id-0-score) and a [Team Score](#sub-id-1-team-score) for every
   player and team not at `0`.

The server leaves out anything at its default. A client that holds packets back while the map loads
keeps them in the order they arrived; "before" and "after" above are that
order. During the game every packet but the Team Count is sent when what it
carries changes, and takes effect at once.

## Sub ID 0: Score

One player's score.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Player ID     | UByte      | `0`     |                                 |
| Score         | LE Int     | `7`     | Signed, absolute.               |

## Sub ID 1: Team Score

One team's score, in any game mode.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet. |
| Team ID       | UByte      | `0`     |                                 |
| Score         | LE Int     | `3`     | Signed, absolute.               |

A client ignores a Score for a player it does not know and a Team Score for a
team the map does not have.

## Sub ID 2: Team Count

How many teams the map is played with. Without one, it is `2`, as in the base
protocol.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet. |
| Teams         | UByte      | `2`     | See below.                      |

| Teams     | Game                                                                             |
|-----------|----------------------------------------------------------------------------------|
| `0`       | `SOLO`: no teams. Every player plays for themselves; the scoreboard is one list. |
| `1`-`254` | That many teams, `0` to `Teams`-1, each with its players and its score.          |
| `255`     | Reserved: never sent. A client ignores the packet.                               |

The extension supports up to 254 teams, `0` to `253`; `255` stays the
spectators. A client that negotiates it accepts any Team Count from `0` to
`254`. How its scoreboard lays out many teams is its own choice.

Every player is on one of the map's teams, or on `255` as a spectator; a client
treats a player on any other team as a spectator. Who can hurt whom stays the
server's choice. In `SOLO` they are on team `0`, which gives them its colour and
nothing else: a client treats every other player as an enemy, with no teammate
name tag or map marker, and team chat reaches whom the server chooses.

Objectives stay the base protocol's, between teams `0` and `1`, so only a map
of two teams or more has them. A team past `1` holds no intel and captures no
territory, and in [CTF State](../protocol075.md#ctf-state),
[TC State](../protocol075.md#tc-state),
[Move Object](../protocol075.md#move-object) and
[Territory Capture](../protocol075.md#territory-capture) a team of `2` still
means neutral. In `SOLO` and with one team, the server still sends the CTF or TC
State the base protocol requires, with no territory in TC, but no
[Intel Pickup](../protocol075.md#intel-pickup),
[Intel Drop](../protocol075.md#intel-drop),
[Intel Capture](../protocol075.md#intel-capture),
[Territory Capture](../protocol075.md#territory-capture) or
[Progress Bar](../protocol075.md#progress-bar).

The Team Count holds for the whole map. A client takes one only if it arrives
between Map Start and State Data, and ignores it otherwise.

## Sub ID 3: Team Info

Gives one team its colour, name and shown events.

| Field Name    | Field Type | Example | Notes                                     |
|---------------|------------|---------|-------------------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                            |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet.           |
| Team ID       | UByte      | `2`     | One of the map's teams.                   |
| Blue          | UByte      | `0`     |                                           |
| Green         | UByte      | `0`     |                                           |
| Red           | UByte      | `255`   |                                           |
| Events        | UByte      | `0`     | Bitmask, see [Team Events](#team-events). |
| Name          | String     | `"Red"` | NUL-terminated.                           |

The name is encoded as a [Chat Message](../protocol075.md#chat-message) is: Code
Page 437, or UTF-8 behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). It
is not limited to the 10 characters of
[State Data](../protocol075.md#state-data), and may be empty. A client ignores a
Team Info for a team the map does not have.

Teams may share a colour or a name. Only the Team Count says who is a teammate;
a client does not infer `SOLO` from two teams looking alike.

### Defined teams

Every team has a colour, a name and events. Teams `0` and `1` take their colour
and name from State Data and start with events `0`; a Team Info for either
overrides them, in whichever order the two arrive, until the next Map Start.
Every other team of the map gets its Team Info in step 1 of the
[Sequence](#sequence), before any player can be in it. A server refuses a
[Change Team](../protocol075.md#change-team) to a team the map does not have,
leaving the player where they were.

### Team Events

Which events a client shows about a team's players. A set bit hides them; `0`
shows everything.

| Bit | Name            | Hides                                                                                                        |
|-----|-----------------|--------------------------------------------------------------------------------------------------------------|
| 0   | `HIDE_PRESENCE` | Join and leave notifications for a player of the team, and team change notifications, see below. |
| 1   | `HIDE_KILLFEED` | The kill feed line of a [Kill Action](../protocol075.md#kill-action) whose killer or victim is of the team.  |
| 2-7 | reserved        | Sent as `0`, ignored by the client.                                                                          |

A team change is hidden only if both the team left and the team joined hide
presence: moving between a shown and a hidden team, either way, is shown.
Spectators have no mask and count as shown.

The mask never hides what a client is shown about its own player: its own join,
kills and deaths are shown whatever its team's mask.

## Sub ID 4: Listed

Puts a player on the scoreboard or takes them off it. Until the first one, every
player is listed.

| Field Name    | Field Type | Example | Notes                                       |
|---------------|------------|---------|---------------------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                              |
| Sub Packet ID | UByte      | `4`     | Always `4` for this sub-packet.             |
| Player ID     | UByte      | `0`     |                                             |
| Listed        | UByte      | `0`     | `0` off the scoreboard. Any other value on. |

A player off the scoreboard is left out of it and of any count of players it
shows, and no
join, team change or leave notification is shown for them, whatever their
team's events. Their kills and deaths still reach the kill feed, as their
team's events allow. They are drawn, heard and hit as any other player, and
their scores are still sent.

Listed hides one player whatever their team; [Team Events](#team-events) hide
what a whole team does. A notification is shown only if neither hides it.

For a player joining during the game, the server sends Listed before the
[Existing Player](../protocol075.md#existing-player) or
[Create Player](../protocol075.md#create-player) that introduces them, so the
join is quiet. A client always lists its own player.

## Sub ID 5: Target Score

The score that wins: a team's, or in `SOLO` a player's. One target applies to
all of them. It takes precedence over the capture limit in
[CTF State](../protocol075.md#ctf-state); until the first one, that capture
limit applies in CTF and there is no target in TC.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x77`  | Always `0x77`.                  |
| Sub Packet ID | UByte      | `5`     | Always `5` for this sub-packet. |
| Target        | LE Int     | `10`    | Signed. `0` or less: no target. |

A team or player reaches the target at a score at or above it. Every score
starts at `0`, so a target of `0` or less would be reached at once; it means
there is none instead.

A client shows the target and never ends the game on it. The server decides the
win and announces it as it does today: the winning flag of
[Intel Capture](../protocol075.md#intel-capture) in CTF, and its own means, such
as chat, otherwise. This extension carries no win.

## Scores

A client never changes a score itself, on
[Kill Action](../protocol075.md#kill-action),
[Intel Capture](../protocol075.md#intel-capture),
[Territory Capture](../protocol075.md#territory-capture) or any other event. The
server sends every change. The scores in
[Existing Player](../protocol075.md#existing-player),
[CTF State](../protocol075.md#ctf-state) and
[Player Properties](player-properties.md) are ignored: this extension is the only
source of scores.

## Reset

[Player Left](../protocol075.md#player-left) clears that player's score and
lists the id again, after the leave is handled, so an unlisted player leaves
silently.

[Map Start](../protocol075.md#map-start-075) clears every player and team score
and the target, sets the Team Count back to `2`, drops every Team Info, and
lists every player again. The [Sequence](#sequence) that follows restores what
the server still wants.

[Create Player](../protocol075.md#create-player) changes none of it: scores, team
info, team count, target and listing all survive a respawn.

See [Extensions](extension.md) for how the extension is negotiated.
