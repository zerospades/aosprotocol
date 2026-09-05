# Teamplay

Gathers the features a team needs to play together and a client cannot provide on
its own: seeing teammates through walls, marking a place in the world, being
pointed at a player by the server, and telling one player something without
telling the rest. Each client-side feature is permitted independently by the
server through a feature bitmask, so a server can enable any combination (or
none) of them.

| ------------: | ------------- |
| Extension ID: | 2             |
| Packet ID:    | 66            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

The extension id, the number carried by `ExtInfo`, is `2`; the packet id is
`64 + extension id` as described in [Extension IDs](extension.md#extension-ids).

### Sub Packets:

| Sub ID | Name     | Direction         | Size |
|--------|----------|-------------------|------|
| 0      | Config   | Server -> Client  | 3    |
| 1      | Ping     | Client <-> Server | 24+  |
| 2      | ESP Mark | Server -> Client  | 13+  |

A client sends one of these and no other: a **Ping** to point at a place. Config
and ESP Mark are the server speaking; a server that receives either from a client
drops it. Whatever a client sends is a request: the server decides whether it
happens at all, who it reaches, and how it is presented.

## Sub ID 0: Config

The server announces which features are permitted. `TEAM_ESP` and `COMPASS_HUD`
are client-side features — the client draws them itself, out of what it already
knows — so they exist only where the server allows them, and a client must not
turn either on by itself. `PING` is the same permission for sending. The server
MAY send this sub-packet at any time; the client applies the new bitmask
immediately (for example to change policy on a map rotation). A feature is only
available to the client while its bit is set.

| Field Name    | Field Type | Example | Notes                                              |
|---------------|------------|---------|----------------------------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                                       |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.                    |
| Features      | UByte      | `0b110` | Feature bitmask, see below.                        |

Feature bitmask:

| Bit | Name           | Meaning                                                                 |
|-----|----------------|-------------------------------------------------------------------------|
| 0   | `TEAM_ESP`     | Client may render teammates through walls, in their team colour, see [What ESP renders](#what-esp-renders). |
| 1   | `PING`         | Client may send Ping packets.                                           |
| 2   | `COMPASS_HUD`  | Client may draw a compass HUD.                                          |
| 3-7 | reserved       | Must be `0`. Clients must ignore unknown bits.                          |

`TEAM_ESP` only renders teammate positions the client already receives from the
base protocol, so it discloses nothing new; the bit is a fair-play policy toggle,
not a data gate.

`COMPASS_HUD` is the same kind of switch: it says whether the client may show a
compass at all, and a client that has none simply never draws on it. It is not a
say in what goes on the compass — each [Ping](#sub-id-1-ping) and
[ESP Mark](#sub-id-2-esp-mark) names its own surfaces, see
[Surfaces](#surfaces) — only in whether the surface exists.

A compass shows the game mode's objectives by default, with no packet asked for
them: the tents and the intel in CTF, the territories in TC, from the positions
the base protocol already sends in
[CTF State](../protocol075.md#ctf-state), [TC State](../protocol075.md#tc-state)
and [Move Object](../protocol075.md#move-object). They are on the minimap already,
so the compass discloses nothing new by bearing them too — it is the same
information, read by direction instead of by position. Everything else on the
compass gets there because a ping or a mark asked for it.

`PING` governs sending, nothing else. With it clear the client sends no pings and
the server ignores any it receives; the server's own pings and marks are
unaffected, and where they are drawn stays the business of the packets that carry
them.

Apart from the compass, no bit stops a client from rendering what the server
sends it: pings and marks are drawn on the surfaces they name, and a server that
wants one unseen does not send it.

After the server changes the config, a Ping the client sent before receiving the
new bitmask may still be in transit. The server simply drops such pings; this is
not a protocol violation and needs no special handling.

## Durations

Pings and ESP marks both carry a lifetime, encoded the same way: an `LE float32`
number of seconds, counted by the client from the moment it receives the packet.

| Value             | Meaning                                                             |
|-------------------|---------------------------------------------------------------------|
| `0`               | Remove: clears the ping or the mark this packet refers to.          |
| positive, finite  | Lifetime in seconds, after which the client removes it by itself.   |
| `+inf` (`0x7F800000`) | Stays until the server removes it or the target leaves.         |
| negative, NaN     | Invalid. The receiver drops the packet.                             |

A float because both ends of the range are real: a spotting ping worth 1.5
seconds and an objective marker worth an hour are written the same way, with no
unit to agree on and no rounding.

The point of the encoding is to let a server pick how much work it wants to do. A
server with nothing special in mind sends a finite duration and forgets the whole
thing: no timer, no removal packet, no state. A server that wants a marker to
follow its own logic sends `+inf` and removes it when its logic says so, which is
the on/off behaviour with no extra packet type. Both spellings cost the same four
bytes.

A server keeping `+inf` marks must remember them anyway, if only to send them to
players who join later; a server using finite durations should re-send an active
mark to a joining client with the time that is left, not the time it started
with. Pings are not re-sent: they are events, not state, and a player who was not
there when the ping was made has nothing to catch up on.

Expiry itself is always client-side. A client may cap how many pings and marks it
displays at once, dropping the oldest first, and is never obliged to render an
absurd number of them.

## Surfaces

Pings and ESP marks both carry a `UByte` saying where the client is to show
them. There are three surfaces, and any combination of them is valid.

| Bit | Name      | Shows                                                                       |
|-----|-----------|-------------------------------------------------------------------------------|
| 0   | `WORLD`   | In the world in 3D: a marker at the position, or the body outline of a mark.   |
| 1   | `MINIMAP` | On the minimap, at the position.                                              |
| 2   | `COMPASS` | On the compass HUD, as a bearing — the direction only, not the place.         |
| 3-7 | reserved  | Must be `0`. Clients must ignore unknown bits.                                |

The packet decides. The client draws it on exactly the surfaces it names, and on
no others; the only surface it may withhold is the compass, when `COMPASS_HUD` is
clear or it has no compass to draw on. A Surfaces of `0` names nothing and asks
for the client's own default placement, which is what a server with no opinion
sends.

Choosing per packet is the point. The same server can put a spotting ping in the
world and on the minimap, a rally marker on the minimap alone, and a gunshot on
the compass alone — direction is all a sound tells you, so a bearing is the
honest way to show it, and a scripted event or an explosion works the same way.
Marks read the same three surfaces: an outline through walls, a dot on the map, a
bearing on the compass, or any mix of the three.

The compass carries no distance and no position, only which way to turn. That is
why it is the surface for anything known by direction alone, and why it stays
readable for a player who has no line of sight at all.

## Colours

Pings and ESP marks both carry their colour as three `UByte` channels, in the
`Blue`, `Green`, `Red` order the base protocol already uses in
[Set Colour](../protocol075.md#set-colour) and
[State Data](../protocol075.md#state-data).

**The client draws the colour the packet carries and assumes nothing.** A client
may adjust a colour for legibility — but must not substitute an unrelated one.
Working out what colour a thing should be is the server's job.

What a colour means is the server's business too, and nothing in this extension
ties it to the Reason: a client is told which colour to draw, never what it
stands for.

The colour applies on every surface the packet names: the marker or outline in
the world, the dot on the minimap, the bearing on the compass. Whether a label
drawn beside it takes the colour too is the client's call.

## Sub ID 1: Ping

A single packet used in both directions. A client sends it to ping the world
position its crosshair points at, with the label it wants; the server validates
all of that and relays it to the players of its
choice, filling in the originating Player ID. The server may also originate a
Ping on its own — objective markers, scripted events, admin callouts — with no
client involved. The position uses the same coordinate frame and `LE float32`
encoding as the base protocol position packets. The client's X/Y/Z coordinates
are required: the server uses them to validate that the client and server agree
on the target's position, preventing desync before performing its own raycasting
validation to confirm line-of-sight.

| Field Name    | Field Type | Example     | Notes                                              |
|---------------|------------|-------------|----------------------------------------------------|
| Packet ID     | UByte      | `66`        | Always `66`.                                       |
| Sub Packet ID | UByte      | `1`         | Always `1` for this sub-packet.                    |
| Player ID     | UByte      | `0`         | On relay, the player that pinged. `255` means the ping originated from the server itself. Ignored on the client -> server direction; the server fills it in authoritatively. |
| X             | LE float32 | `256.0`     | World X coordinate.                                |
| Y             | LE float32 | `256.0`     | World Y coordinate.                                |
| Z             | LE float32 | `40.0`      | World Z coordinate.                                |
| Duration      | LE float32 | `5.0`       | Display time, see [Durations](#durations). Server -> client only: a client sends `0` here and the server, which is authoritative, fills it in. |
| Surfaces      | UByte      | `0b011`     | Where the client shows it, see [Surfaces](#surfaces). Server -> client only: a client sends `0` and the server decides. |
| Blue          | UByte      | `0`         | Marker colour, blue channel, see [Ping colour](#ping-colour). |
| Green         | UByte      | `0`         | Marker colour, green channel.                      |
| Red           | UByte      | `255`       | Marker colour, red channel.                        |
| Message ID    | UByte      | `0`         | Reserved and unimplemented. Must be `0` in version 1, see below. |
| Reason        | UTF-8 text | `""`        | Free-form label, the remaining bytes of the packet. |

A player has one active ping at a time: a new ping from the same Player ID
replaces the previous one and restarts its lifetime, and a Duration of `0`
removes it without placing another. The server's own pings (Player ID `255`)
behave the same way, so a server that wants several permanent markers at once
needs ESP marks or a ping per originating id, not several pings from `255`.

**A client shows who pinged.** The ping carries the name the client already has
for the player in Player ID, so a marker is never anonymous and a label is never
read as coming from somebody who did not write it. Where the name goes — beside
the marker, under it, in the chat log — is the client's business; that it is
there is not.

**A Player ID of `255` is the exception: the client shows no sender at all.**
There is no player behind it, `255` is not an id it may look up, and the ping is
never attributed to whoever happens to hold a nearby id. It stands on its own,
with its label if it has one.

A ping always points somewhere: there is no way to send one that means something
without meaning it about a place.

A ping carries no audience. The client has no way to ask for team only, for the
whole server, or for one player, because it has no business knowing: it points at
a place and the server decides who is shown it. The same is true of the label —
a client attaches the label it likes, and the server keeps it, replaces it or
drops the ping entirely.

The label of a ping is a free string. **Message ID** is reserved and
unimplemented: version 1 sends `0`, and a receiver that gets anything else
renders the packet and ignores the byte. The byte is here so that a later version
can name a label instead of spelling it, without moving where the Reason begins
or changing how long it is. It carries the name it will have then, and reserving
it now is what keeps that version additive.

The Reason occupies the remaining bytes of the packet (no length prefix or
terminator); its length is implied by the packet length, and it may be empty,
which is the shortest form of the packet at 24 bytes and a neutral "look here"
marker. It is a free-form, client-defined UTF-8 string, consistent with the
[UTF-8 Chat](utf-8-chat.md) convention. Before relaying, the server must validate
it as well-formed UTF-8 and drop (or sanitise) anything malformed rather than
forwarding bytes verbatim. The server should cap its length, truncating on a
codepoint boundary if needed. Clients render the string as received and fall back
to a neutral marker for anything they do not recognise.

A Reason says anything a player wants said — a place name, a plan, a joke. The
cost is that **free text falls out of translation**: it travels as the bytes the
sender wrote, and every reader sees those same bytes in the sender's language,
however many of them speak it. A server that wants a room where everybody
understands everybody can clear the Reason and relay the marker alone.

A dead player does not ping. A client must not send a Ping while its player is
dead, and the server drops any it receives from a dead player, exactly as it
drops one sent while the feature is switched off. Waiting to respawn is not a
vantage point, and a corpse pointing at things the living cannot see is a way of
spectating the enemy. Whether spectators may ping is the server's call.

To identify the target of a ping, the server performs a raycasting check from the
client's position through their crosshair direction using the X/Y/Z coordinates
sent by the client to validate sync and confirm line-of-sight, rather than
relying on client data to identify the target.

Server handling of a client -> server Ping:

* The server should rate-limit requests (a sane default is at most one per player
  per second).
* The label a client asks for is a request like the rest: the server may replace
  it, empty it, or drop the ping over it.
* Whatever a client puts in Duration, Surfaces and the colour bytes is ignored,
  and is never a reason to drop the ping: the server overwrites those fields with
  its own values and relays it as usual.
* The server must validate the coordinates by comparing them against its own state
  for the pinged player (at least the map bounds of 512 x 512 x 64, and the
  player's actual position for sync checking). The server should reject pings to
  positions that diverge significantly from its own tracked state.
* Who receives the relayed Ping is left to the server's discretion (team only,
  everyone, spectators, etc.); the protocol does not mandate a distribution
  policy. Relaying it to nobody is a valid decision too.

How long a ping stays is the server's call alone, never the pinging client's:
Duration is only meaningful on the relay. A client asks for a ping, the server
decides what that ping is worth — `5.0` seconds is the conventional value and
what a server with no opinion should send, while a server building something of
its own can make a ping last a round or vanish in half a second, without the
client needing to know that policy or the server needing a removal packet.

### Ping colour

The colour follows the rules in [Colours](#colours), and **only the server sets
it**: whatever a client puts in the three bytes is ignored and the server fills
them in itself. A server with no opinion sends the team colour of the player who
pinged, and has no policy to write.

## Sub ID 2: ESP Mark

Marks a player as visible through walls to whoever receives the packet. Unlike
`TEAM_ESP`, which lets a client reveal its own team on its own initiative, a mark
is decided by the server, targets one player, and is unrelated to teams: the
server can reveal a player to their own team, to the other team, to everyone, or
to a single player. That covers spotting the scoreboard leader, showing a hunted
player to the hunters, exposing a cheater to the whole server, or a game mode that
reveals a carrier while they hold the intel. The mark carries the colour it is
drawn in, so those cases do not all look the same.

This sub-packet travels **server -> client only**. Unlike the Ping and the
Message, it has no client-sent form: a client never asks for a mark and never
sends sub-packet `2`. A server receiving it from a client must drop it, since the
only thing such a packet can be is an attempt to reveal a player to somebody the
server did not choose.

| Field Name    | Field Type | Example    | Notes                                          |
|---------------|------------|------------|------------------------------------------------|
| Packet ID     | UByte      | `66`       | Always `66`.                                   |
| Sub Packet ID | UByte      | `2`        | Always `2` for this sub-packet.                |
| Player ID     | UByte      | `7`        | The player to reveal.                          |
| Duration      | LE float32 | `10.0`     | Lifetime, see [Durations](#durations).         |
| Surfaces      | UByte      | `0b101`    | Where the client shows it, see [Surfaces](#surfaces). |
| Flags         | UByte      | `0b1`      | Lifetime modifiers, see below.                 |
| Blue          | UByte      | `0`        | Outline colour, blue channel, see [Colours](#colours). |
| Green         | UByte      | `0`        | Outline colour, green channel.                 |
| Red           | UByte      | `255`      | Outline colour, red channel.                   |
| Message ID    | UByte      | `0`        | Reserved, as on the [Ping](#sub-id-1-ping). Must be `0`. |
| Reason        | UTF-8 text | `"leader"` | Free-form label, the remaining bytes of the packet. |

Flags:

| Bit | Name               | Meaning                                                                     |
|-----|--------------------|-----------------------------------------------------------------------------|
| 0   | `CLEAR_ON_RESPAWN` | The mark ends the next time the marked player spawns.                       |
| 1   | `SHOW_NAME`        | The client shows the marked player's name. Clear, it shows the outline alone.|
| 2-7 | reserved           | Must be `0`. Clients must ignore unknown bits.                              |

`CLEAR_ON_RESPAWN` is keyed to the spawn rather than to the death: any
[Create Player](../protocol075.md#create-player) for that id ends the mark,
whether the player was killed, changed team, changed weapon or was moved by a
script, and the client needs no death bookkeeping to implement it. A mark on a
player who dies and stays dead lasts until they come back, which is what
"reveal them until they get away" wants anyway.

`SHOW_NAME` is set or clear on every mark, and the client obeys it: the name is
shown when it is set and not shown when it is clear. Only the server knows what a
mark is meant to reveal, so the choice is never the client's.

Duration and the lifetime flag are independent, so every lifetime a server is
likely to want falls out of the same five bytes:

| Intent                                      | Duration | Flags              |
|---------------------------------------------|----------|--------------------|
| Reveal for a while                          | `3.5`    | `0`                |
| Reveal until the server clears it           | `+inf`   | `0`                |
| Reveal until they respawn                   | `+inf`   | `CLEAR_ON_RESPAWN` |
| Reveal for a while, or until they respawn   | `3.5`    | `CLEAR_ON_RESPAWN` |
| Clear the mark now                          | `0`      | `0`                |

`SHOW_NAME` is orthogonal to all of these and may be set with any of them.

The label works exactly as on a Ping, Message ID reserved and Reason following
the same rules — remaining bytes of the packet, no length prefix or terminator,
validated by the server as well-formed UTF-8, capped and truncated on a codepoint
boundary — and the client falls back to a neutral highlight for anything it does
not recognise. An empty Reason is valid and is the common case: the packet is
then 13 bytes and the client shows the player highlighted with no label.
`"cheater"`, `"leader"` and `"carrier"` are examples, not assigned values.

The audience is the set of clients the server sends the packet to; there is no
audience field. A field would have to be enforced by the client, and a client
that ignores it would reveal players it was never meant to see — the same reason
the Ping relay leaves distribution to the server. It also means a player is not
told they are being revealed to others unless the server includes them in the
recipients, which is what the punishment case wants; a client that receives a
mark for its own player id may show it as a "you are marked" indicator.

A mark is an instruction rather than a permission, so the client renders it
whatever the `TEAM_ESP` bit says, and marks are not gated by the Config bitmask.
A client with `TEAM_ESP` clear still shows a marked teammate. When the bit is set
and the mark lands on a teammate the client is already revealing, the two are
both shown, see [A marked teammate](#a-marked-teammate).

### What ESP renders

The two paths are not held to the same rendering, because one is the client
showing its own team and the other is the server pointing at somebody.

`TEAM_ESP`, while its bit is set, reveals **the client's own teammates, drawn in
their team's colour** — the one the server sent in
[State Data](../protocol075.md#state-data) for the team that player is on. That is
the whole of what the bit turns on: teammates, in the team colour, so the
highlight reads as team information at a glance and is never mistaken for
anything the server said. It reveals nobody else; an enemy is shown through a
wall only when the server marks them.

The shape is left to the client: a body outline, a box around the player, a
chevron above them, a dot on the edge of the screen — whatever the client
considers good practice — as long as the colour is the team's. Showing the name
alongside is recommended and also the client's call.

An **ESP Mark that names the `WORLD` surface must draw the player's body outline
through walls**, following the body and its pose rather than standing in for it
with a box, a dot or a floating marker. The server marked one specific player,
and the audience has to see where exactly they are, not merely that somebody is
around there. The outline is drawn in the mark's [colour](#colours). On the
minimap the mark is a dot at the player's position, on the compass a bearing
towards them.

Whether the name goes with it is the server's decision, carried by the
`SHOW_NAME` flag: set, the client shows the marked player's name; clear, it shows
the outline and nothing else.

#### A marked teammate

The two paths can land on the same player: a teammate the client is already
revealing under `TEAM_ESP`, who the server then marks. Both colours are then
true — the team colour says they are yours, the mark colour says what the server
is telling you about them — and one drawn over the other would silently throw the
other away.

So the client **blinks between the two colours**, alternating between the team
colour and the mark colour, each shown for a roughly equal share of the cycle,
on every surface the mark names: the outline in the world, the dot on the
minimap, the bearing on the compass. A period of about one second is the
recommended default; a client may tune it, but it must be slow enough to read
both colours and fast enough that a glance catches both. Nothing else changes as
it blinks: the outline stays the body outline the mark requires, in the same
place, at the same thickness, and the dot and the bearing stay where they are.

This applies only while both are actually in effect. With `TEAM_ESP` clear, a
marked teammate is drawn in the mark colour alone, since the client is not
revealing teammates at all and there is no second colour to show — a mark is an
instruction rather than a permission, and it is rendered whatever the bit says.
A marked enemy never blinks either, for the same reason: no team colour of the
viewer's is in play. A server that marks a teammate in their own team colour gets
no blink worth looking at, the two colours being the same, and should pick
another colour when it wants the mark seen.

The rest is the client's business in every case: outline thickness, opacity,
whether the highlight fades with distance, and how the label is rendered beside
it, if at all.

Marks are state held per player id, and one mark per player: a new mark replaces
the previous one and restarts its timer. A mark is dropped when its Duration
expires, when a mark with Duration `0` arrives for that id, when its target
spawns and `CLEAR_ON_RESPAWN` is set, or under the general rules of
[Per-player state](#per-player-state). Without `CLEAR_ON_RESPAWN` it survives
death and respawn, so a punishment mark does not have to be re-sent every time
its target is killed.

## Per-player state

Everything this extension creates belongs to a player id, and all of it is freed
the moment that player leaves. When a client receives
[Player Left](../protocol075.md#player-left) for an id it drops, for that id: the
player's active ping and the ESP mark on them if any. The server drops the same
things on its side and stops referring to the id.

The reason is that ids are recycled. A mark or a ping outliving its owner does
not fade away quietly — it lands on whoever takes the id next, and that player is
suddenly revealed to the enemy team, or pinned to a marker they never made.

A map change ([Map Start](../protocol075.md#map-start-075)) clears everything for
every id, on both ends. Nothing this extension holds is meant to survive a world.

See [Extensions](extension.md) for how the extension is negotiated.
