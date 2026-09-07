# Teamplay

The features a team needs to play together and a client cannot provide on its
own: seeing teammates through walls, marking a place in the world, and being
pointed at a player by the server. Each client-side feature is permitted
independently by the server, so a server can enable any combination or none.

| ------------: | ------------- |
| Extension ID: | 2             |
| Packet ID:    | 66            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

The packet id is `64 + extension id`, see
[Extension IDs](extension.md#extension-ids).

### Sub Packets:

| Sub ID | Name     | Direction         | Size |
|--------|----------|-------------------|------|
| 0      | Config   | Server -> Client  | 11   |
| 1      | Ping     | Client <-> Server | 24+  |
| 2      | ESP Mark | Server -> Client  | 13+  |

Config and ESP Mark are server to client; a server that receives either from a
client drops it. Ping is the only packet a client sends, and what it sends is a
request: the server decides whether it happens, who it reaches, and how it looks.

## Sub ID 0: Config

Which features are permitted, and which way north is.

| Field Name    | Field Type | Example | Notes                              |
|---------------|------------|---------|------------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                       |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.    |
| Features      | UByte      | `0b110` | Bitmask, see below.                |
| North X       | LE float32 | `0.0`   | See [North](#north).               |
| North Y       | LE float32 | `-1.0`  |                                    |

Always 11 bytes.

| Bit | Name          | Meaning                                                  |
|-----|---------------|----------------------------------------------------------|
| 0   | `TEAM_ESP`    | Client may render teammates through walls, in their team colour. See [What ESP renders](#what-esp-renders). |
| 1   | `PING`        | Client may send Ping packets.                            |
| 2   | `COMPASS_HUD` | Client may draw a compass HUD.                           |
| 3-7 | reserved      | Must be `0`. Clients **must** ignore unknown bits.                |

The server **must** send a Config once the extension is negotiated, and may send
further ones at any time. The client applies each one immediately and in full; a
feature is available only while its bit is set, and the packet stands until
another replaces it. A client must not turn `TEAM_ESP` or `COMPASS_HUD` on by
itself.

`TEAM_ESP` only draws teammate positions the client already receives, so it
discloses nothing new — the bit is a fair-play toggle, not a data gate.
`COMPASS_HUD` says whether a compass exists, not what goes on it; each
[Ping](#sub-id-1-ping) and [ESP Mark](#sub-id-2-esp-mark) names its own
[surfaces](#surfaces). A compass shows the mode's objectives by default with no
packet asked for them, from
[CTF State](../protocol075.md#ctf-state), [TC State](../protocol075.md#tc-state)
and [Move Object](../protocol075.md#move-object) — the same information the
minimap already carries, read by direction instead of by position. Everything
else on the compass gets there because a ping or a mark asked for it.

`PING` governs sending, nothing else. With it clear the client sends none and the
server ignores any it receives, including ones already in transit when the config
changed — that is not a protocol violation and needs no special handling. The
server's own pings and marks are unaffected, and where they are drawn stays the
business of the packets that carry them.

The Config belongs to the connection, so
[Map Start](../protocol075.md#map-start-075) does not clear it — bitmask and
north both carry into the new world, and a server whose maps differ sends a new
Config after the change. Everything else this extension holds is dropped, see
[Per-player state](#per-player-state). A Config arriving during map transfer
refers to no player and no position, so the client applies it as it arrives.

### North

The map-plane components of a vector pointing north, same coordinate frame and
encoding as the base protocol's position packets. The base protocol names no
orientation, so every client that ever drew a compass had to hardcode one; these
two fields make it the server's to state.

A vector and not an angle, because an angle needs a convention agreed in advance
— which axis is zero, which way it grows, degrees or radians — and each is a way
to disagree silently. The server should send a unit vector and the client
normalises whatever arrives, since only the direction is used.

`(0, 0)`, a NaN or an infinity in either component is malformed, not a value: the
client falls back to `(0, -1)` and applies the rest of the packet normally.

North is carried whether or not `COMPASS_HUD` is set, so a client handed the bit
mid-round already has the direction to point. A server changing only policy
repeats the north it already gave — a compass that turns under a player undoes
every bearing they have been given. North itself is free to change: nothing here
stops a server moving it between worlds or during a round, and a mode may want
exactly that.

## Durations

Pings and marks both carry an `LE float32` number of seconds, counted by the
client from the moment it receives the packet.

| Value                 | Meaning                                                |
|-----------------------|--------------------------------------------------------|
| `0`                   | Remove the ping or mark this packet refers to.         |
| positive, finite      | Lifetime in seconds; the client removes it itself.     |
| `+inf` (`0x7F800000`) | Stays until the server removes it or the target leaves.|
| negative, NaN         | Invalid. The receiver drops the packet.                |

A float because both ends of the range are real: a 1.5-second spotting ping and
an hour-long objective marker are written the same way. The encoding lets a
server choose how much work it does — a finite duration needs no timer, no
removal packet and no state, while `+inf` gives on/off behaviour with no extra
packet type.

A server keeping `+inf` marks must remember them to send to players who join
later; one using finite durations should re-send an active mark to a joining
client with the time that is left. Pings are not re-sent — they are events, not
state. Expiry is always client-side, and a client may cap how many it displays at
once, dropping the oldest first.

## Surfaces

A `UByte` saying where the client shows the ping or mark. Any combination is
valid.

| Bit | Name      | Shows                                                       |
|-----|-----------|-------------------------------------------------------------|
| 0   | `WORLD`   | In 3D: a marker at the position, or a mark's body outline.  |
| 1   | `MINIMAP` | On the minimap, at the position.                            |
| 2   | `COMPASS` | On the compass, as a bearing — direction only, not place.   |
| 3-7 | reserved  | Must be `0`. Clients **must** ignore unknown bits.                   |

The client draws it on exactly the surfaces named and no others. The only one it
may withhold is the compass, when `COMPASS_HUD` is clear or it has none. A
Surfaces of `0` asks for the client's default placement, which is what a server
with no opinion sends.

Choosing per packet is the point: a spotting ping in the world and on the
minimap, a rally marker on the minimap alone, a gunshot on the compass alone —
direction is all a sound tells you. Bearings are read against the
[north](#north) the server last sent.

## Colours

Three `UByte` channels in the `Blue`, `Green`, `Red` order the base protocol
already uses in [Set Colour](../protocol075.md#set-colour).

The client draws the colour the packet carries and assumes nothing. It may adjust
for legibility but must not substitute an unrelated colour. What a colour means
is the server's business, and nothing here ties it to the Reason. The colour
applies on every surface the packet names; whether a label drawn beside it takes
the colour too is the client's call.

## Sub ID 1: Ping

Points at a world position. A client sends one for the position its crosshair
points at; the server validates it and relays it to the players of its choice,
filling in the originating Player ID. The server may also originate one with no
client involved. The position uses the same coordinate frame and `LE float32`
encoding as the base protocol's position packets.

| Field Name    | Field Type | Example | Notes                                     |
|---------------|------------|---------|-------------------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                              |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.           |
| Player ID     | UByte      | `0`     | On relay, the player that pinged; `255` means the server itself. Ignored client -> server; the server fills it in. |
| X             | LE float32 | `256.0` | World coordinate.                         |
| Y             | LE float32 | `256.0` |                                           |
| Z             | LE float32 | `40.0`  |                                           |
| Duration      | LE float32 | `5.0`   | See [Durations](#durations). A client sends `0`; the server fills it in. |
| Surfaces      | UByte      | `0b011` | See [Surfaces](#surfaces). A client sends `0`; the server decides. |
| Blue          | UByte      | `0`     | See [Ping colour](#ping-colour).          |
| Green         | UByte      | `0`     |                                           |
| Red           | UByte      | `255`   |                                           |
| Message ID    | UByte      | `0`     | Reserved. Must be `0` in version 1.       |
| Reason        | UTF-8 text | `""`    | Free-form label, the remaining bytes.     |

One active ping per Player ID: a new one replaces the previous and restarts its
lifetime, and a Duration of `0` removes it without placing another. A server
wanting several permanent markers uses ESP marks, not several pings from `255`.

**A client shows who pinged**, using the name it already has for Player ID, so a
label is never read as coming from somebody who did not write it. Where the name
goes is the client's business; that it is there is not. **Player ID `255` is the
exception** — no sender is shown, and the ping is never attributed to whoever
holds a nearby id.

A ping carries no audience, and no way to ask for team-only or whole-server,
because the client has no business knowing: it points at a place and the server
decides who sees it.

**Message ID** is reserved and unimplemented; a receiver that gets a non-zero
value renders the packet and ignores the byte. It exists so a later version can
name a label instead of spelling it, without moving where the Reason begins.

The **Reason** is the remaining bytes of the packet — no length prefix or
terminator — and may be empty, which is the shortest form at 24 bytes and a
neutral "look here" marker. It is client-defined UTF-8, consistent with
[UTF-8 Chat](utf-8-chat.md). The server must validate it as well-formed UTF-8 and
drop or sanitise anything malformed rather than forwarding bytes verbatim, and
should cap its length, truncating on a codepoint boundary. Clients render it as
received, and fall back to a neutral marker for anything they do not recognise.

Free text falls out of translation: every reader sees the sender's language,
however many of them speak it. A server that wants a room where everybody
understands everybody clears the Reason and relays the marker alone.

A dead player does not ping. The client must not send one while dead and the
server drops any it receives from a dead player. Whether spectators may ping is
the server's call.

Server handling of a client -> server Ping:

* The server **should** rate-limit requests; at most one per player per second is
  a sane default.
* The server identifies the target by raycasting from the client's position
  through their crosshair direction, using the X/Y/Z the client sent to check
  that the two agree and to confirm line-of-sight, rather than trusting the
  client to name it.
* The server **must** validate the coordinates against its own state for that
  player — at least the map bounds of 512 x 512 x 64, and the player's actual
  position — and **should** reject pings to positions that diverge significantly
  from it.
* Whatever the client put in Duration, Surfaces and the colour bytes is ignored
  and overwritten; this is never a reason to drop the ping.
* The label is a request like the rest: the server may replace it, empty it, or
  drop the ping over it.
* Who receives the relay is the server's discretion. Relaying to nobody is a
  valid decision.

`5.0` seconds is the conventional Duration and what a server with no opinion
sends.

### Ping colour

Follows [Colours](#colours), and **only the server sets it** — whatever a client
puts in the three bytes is ignored. A server with no opinion sends the team
colour of the player who pinged.

## Sub ID 2: ESP Mark

Marks one player as visible through walls to whoever receives the packet. Unlike
`TEAM_ESP` it is decided by the server and unrelated to teams: the server can
reveal a player to their own team, the other team, everyone, or one person.

Server to client only. A client never asks for a mark and never sends sub-packet
`2`; a server receiving one drops it, since the only thing such a packet can be
is an attempt to reveal a player to somebody the server did not choose.

| Field Name    | Field Type | Example    | Notes                                 |
|---------------|------------|------------|---------------------------------------|
| Packet ID     | UByte      | `66`       | Always `66`.                          |
| Sub Packet ID | UByte      | `2`        | Always `2` for this sub-packet.       |
| Player ID     | UByte      | `7`        | The player to reveal.                 |
| Duration      | LE float32 | `10.0`     | See [Durations](#durations).          |
| Surfaces      | UByte      | `0b101`    | See [Surfaces](#surfaces).            |
| Flags         | UByte      | `0b1`      | See below.                            |
| Blue          | UByte      | `0`        | Outline colour, see [Colours](#colours). |
| Green         | UByte      | `0`        |                                       |
| Red           | UByte      | `255`      |                                       |
| Message ID    | UByte      | `0`        | Reserved, as on the Ping. Must be `0`.|
| Reason        | UTF-8 text | `"leader"` | Free-form label, the remaining bytes. |

| Bit | Name               | Meaning                                              |
|-----|--------------------|------------------------------------------------------|
| 0   | `CLEAR_ON_RESPAWN` | The mark ends the next time the marked player spawns.|
| 1   | `SHOW_NAME`        | The client shows the marked player's name. Clear, it shows the outline alone. |
| 2-7 | reserved           | Must be `0`. Clients **must** ignore unknown bits.            |

`CLEAR_ON_RESPAWN` is keyed to the spawn, not the death: any
[Create Player](../protocol075.md#create-player) for that id ends the mark,
whether the player was killed, changed team, changed weapon or was moved by a
script, so the client needs no death bookkeeping. A mark on a player who stays
dead lasts until they come back.

Duration and the flag are independent, so every lifetime a server is likely to
want falls out of the same five bytes:

| Intent                                    | Duration | Flags              |
|-------------------------------------------|----------|--------------------|
| Reveal for a while                        | `3.5`    | `0`                |
| Reveal until the server clears it         | `+inf`   | `0`                |
| Reveal until they respawn                 | `+inf`   | `CLEAR_ON_RESPAWN` |
| Reveal for a while, or until they respawn | `3.5`    | `CLEAR_ON_RESPAWN` |
| Clear the mark now                        | `0`      | `0`                |

`SHOW_NAME` is orthogonal and may be set with any of them.

The label works as on the Ping: Message ID reserved, Reason the remaining bytes,
server-validated UTF-8, capped and truncated on a codepoint boundary, with the
client falling back to a neutral highlight for anything it does not recognise. An
empty Reason is the common case — the packet is then 13 bytes and the client
shows the player highlighted with no label. `"cheater"`, `"leader"` and
`"carrier"` are examples, not assigned values.

The audience is the set of clients the server sends the packet to; there is no
audience field, because a field would have to be enforced by the client and a
client that ignored it would reveal players it was never meant to. It also means
a player is not told they are marked unless the server includes them in the
recipients, which is what the punishment case wants; a client receiving a mark
for its own id may show a "you are marked" indicator.

A mark is an instruction, not a permission, so it is not gated by the Config
bitmask: a client with `TEAM_ESP` clear still shows a marked teammate.

One mark per player id. A new mark replaces the previous and restarts its timer.
A mark is dropped when its Duration expires, when a Duration of `0` arrives for
that id, when its target spawns and `CLEAR_ON_RESPAWN` is set, or under
[Per-player state](#per-player-state). Without `CLEAR_ON_RESPAWN` it survives
death and respawn, so a punishment mark need not be re-sent on every kill.

### What ESP renders

The two paths are not held to the same rendering: one is the client showing its
own team, the other is the server pointing at somebody.

`TEAM_ESP`, while set, reveals **the client's own teammates in their team
colour** — the one from [State Data](../protocol075.md#state-data). That is the
whole of what the bit turns on, so the highlight reads as team information and is
never mistaken for something the server said. It reveals nobody else. The shape
is the client's call — outline, box, chevron, edge-of-screen dot — as long as the
colour is the team's. Showing the name alongside is recommended and also the
client's call.

An **ESP Mark naming the `WORLD` surface must draw the player's body outline
through walls**, following the body and its pose rather than standing in for it
with a box or a floating marker: the server marked one specific player and the
audience has to see where exactly they are. The outline takes the mark's
[colour](#colours). On the minimap the mark is a dot at the player's position, on
the compass a bearing towards them. Whether the name goes with it is the
`SHOW_NAME` flag's decision, never the client's.

#### A marked teammate

Both paths can land on the same player: a teammate already revealed under
`TEAM_ESP` whom the server then marks. Both colours are true — the team colour
says they are yours, the mark colour says what the server is telling you — and
drawing one over the other throws the other away.

So the client **blinks between the two colours**, each for a roughly equal share
of the cycle, on every surface the mark names. About one second is the
recommended period; a client may tune it, but it must be slow enough to read both
and fast enough that a glance catches both. Nothing else changes: the outline
stays the body outline, in the same place, at the same thickness.

This applies only while both are in effect. With `TEAM_ESP` clear a marked
teammate is drawn in the mark colour alone, and a marked enemy never blinks
either — no team colour of the viewer's is in play.

## Per-player state

Everything this extension creates belongs to a player id and is freed when that
player leaves. On [Player Left](../protocol075.md#player-left) the client drops,
for that id, the player's active ping and any mark on them. The server does the
same and stops referring to the id.

Ids are recycled, and a mark or ping outliving its owner does not fade quietly —
it lands on whoever takes the id next, who is suddenly revealed to the enemy team
or pinned to a marker they never made.

[Map Start](../protocol075.md#map-start-075) clears everything for every id on
both ends. The Config is not held against a player and survives, see
[Sub ID 0: Config](#sub-id-0-config).

See [Extensions](extension.md) for how the extension is negotiated.
