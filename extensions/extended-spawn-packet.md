# Extended Spawn Packet

Carries per-player-id properties the base [Create Player](../protocol075.md#create-player)
packet has no room for. Version 1 carries two: *silence*, players that exist in
the world and are rendered like any other player but take no part in the
presentation *other* clients build around their player list — scoreboard, player
counts, presence notices, kill feed — and a *colour* that marks a player out in
the world in place of their team's. What silence leaves alone is the silent
player's own client.

The base protocol has a single notion of a player: a client only knows about a
player because it received an [Existing Player](../protocol075.md#existing-player)
or [Create Player](../protocol075.md#create-player) packet for it, and that same
list drives the scoreboard and every player-related notification. This extension
extends the spawn packet instead of adding a second entity concept, so
server-side actors (NPCs, RPG mobs, training dummies) can spawn, act, die and
disappear as loudly or as quietly as the server wants, and a server can move a
real player in or out of the game unannounced.

| ------------: | ------------- |
| Extension ID: | 3             |
| Packet ID:    | 67            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

The extension id negotiated in `ExtInfo` is `3`; the packet id is `64 + extension
id` as described in [Extension IDs](extension.md#extension-ids).

### Sub Packets:

| Sub ID | Name                  | Direction        | Size          |
|--------|-----------------------|------------------|---------------|
| 0      | Extended Create Player| Server -> Client | `21+`         |
| 1      | Set Flags             | Server -> Client | `2+2*entries` |
| 2      | Set Colour            | Server -> Client | `2+4*entries` |

Sub packet 0 replaces [Create Player](../protocol075.md#create-player): it
carries the same spawn data plus the properties, so a spawn stays one packet and
cannot be misread by a client that has not been told about them yet. Sub packets
1 and 2 cover everything a spawn cannot: the players already in the world when a
client joins, and any change made while the game runs.

## Flags

Sub packets 0 and 1 carry the same one byte mask.

| Bit | Name             | Meaning                                                                                                                                              |
|-----|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| 0   | `HIDE_ROSTER`    | The player is left out of the scoreboard, of the total / alive / spectator counts, and of spectator follow-camera cycling and similar player pickers. |
| 1   | `HIDE_PRESENCE`  | No join, team change or leave notification is shown for this player.                                                                                 |
| 2   | `HIDE_KILLFEED`  | No kill feed line is shown for a kill this player made or suffered.                                                                                  |
| 3   | `NO_STATS`       | The player is ignored by client-side statistics: kill and death counters, kill streaks, and domination / revenge tracking.                            |
| 4   | `CUSTOM_COLOR`   | The player is drawn in the colour bound to their id, in place of their team colour. See [Colour](#colour).                                            |
| 5-7 | reserved         | Must be `0`. Clients must ignore unknown bits.                                                                                                       |

The bits are independent, which is the point of the mask. An RPG server that
wants its mobs to appear and vanish without a word but still wants kills
announced sets `HIDE_ROSTER | HIDE_PRESENCE | NO_STATS` and leaves
`HIDE_KILLFEED` clear; a fully silent actor sets the first four. `CUSTOM_COLOR`
is unrelated to the others and combines with any of them: an ordinary player
marked out by colour sets it alone.

Presence is a single bit rather than one for joining and one for leaving: a
client that announced neither arrival nor departure is coherent, and one that
announces only the departure of a player it never announced is not.

A mask of `0` means the player is presented normally. That is how a player is
un-silenced — there is no separate clear sub-packet — and it is what a server
sends to reveal an actor that was silent until then, an ambushing NPC turning
into a scoreboard participant for example. Clearing the mask does not discard a
colour bound to the id: it stops the client using it, and the colour is still
there if the bit comes back. Only a spawn,
[Player Left](../protocol075.md#player-left) or
[Map Start](../protocol075.md#map-start-075) discards it.

Every hiding bit — `HIDE_ROSTER`, `HIDE_PRESENCE`, `HIDE_KILLFEED`, `NO_STATS` —
describes what a client shows about **another** player. None of them changes
what a player is shown about themselves: those bits are ignored for the client's
own player id, so a silenced player still sees their own row on the scoreboard,
still counts towards the totals their client displays, still reads their own
kills and deaths in the kill feed and still keeps their own statistics. A client
is never silent to itself, and nothing this extension can set takes information
away from the player it is set on.

`CUSTOM_COLOR` is the exception to that rule, and it is not one of the hiding
bits: it withholds nothing, it changes an appearance, and a player marked out by
colour is marked out on their own screen too.

## Colour

A colour is three bytes, written **blue, green, red**, the order the base
protocol uses in [Set Colour](../protocol075.md#set-colour) and
[Fog Colour](../protocol075.md#fog-colour). Like the mask, a colour is bound to
the player id and outlives the player occupying it.

`CUSTOM_COLOR` is what switches the override on; the colour bytes never carry
that decision themselves. The base protocol has no value meaning *no colour* —
every colour field in 0.75 is unconditionally present and always a real colour,
and `0, 0, 0` is black, a legal block colour and a legal team colour. Spending it
as a sentinel here would cost servers the one colour nothing else can express,
so the flag says whether to override and the bytes say only what to.

**A client that has no colour for an id draws the team colour, even with
`CUSTOM_COLOR` set.** The colour degrades the way the mask does: a flag that
arrives before the colour it refers to, or a client that missed the colour,
produces an ordinary team-coloured player rather than a black one. It also means
a server may set the bit and the colour in either order — setting the bit first
costs at most a frame in the team colour, and nothing that cannot be taken
back.

The override replaces the team colour wherever the client draws the player from
it — the player model, the tool or weapon in their hands, their corpse, their
tracers. It is a colour, not a team: `CUSTOM_COLOR` changes nothing about who is
a teammate, who may be hit, or what the game mode counts, and a client must not
infer a team from it. Where a client colours something by team id rather than by
the player — the scoreboard row, a kill feed name — it may keep using the team
colour; the override is about the player in the world.

## Sub ID 0: Extended Create Player

Spawns a player and sets its flags in the same packet. Sent instead of
[Create Player](../protocol075.md#create-player), on initial spawn and on every
respawn, to clients that negotiated this extension.

| Field Name    | Field Type   | Example  | Notes                                             |
|---------------|--------------|----------|---------------------------------------------------|
| Packet ID     | UByte        | `67`     | Always `67`.                                      |
| Sub Packet ID | UByte        | `0`      | Always `0` for this sub-packet.                   |
| Player ID     | UByte        | `254`    |                                                   |
| Flags         | UByte        | `0b1011` | See [Flags](#flags).                              |
| Weapon        | UByte        | `0`      | As in Create Player.                              |
| Team          | UByte        | `0`      | As in Create Player.                              |
| X position    | LE Float     | `256.0`  | As in Create Player.                              |
| Y position    | LE Float     | `256.0`  | As in Create Player.                              |
| Z position    | LE Float     | `40.0`   | As in Create Player.                              |
| Blue          | UByte        | `160`    | See [Colour](#colour).                            |
| Green         | UByte        | `32`     | See [Colour](#colour).                            |
| Red           | UByte        | `200`    | See [Colour](#colour).                            |
| Name          | CP437 String | `Wolf`   | As in Create Player, same encoding, not `\0` terminated. |

Everything shared with Create Player behaves exactly as it does there, including
the spawn height adjustment clients apply; the additions are Flags and the
colour, which the client stores for the id and applies before it emits anything
about the spawn. The packet is therefore atomic: there is no window in which the
client considers the player an ordinary participant, or draws them in the wrong
colour.

The three colour bytes are always present, whether or not `CUSTOM_COLOR` is set,
and they sit before the Name because the name has no length of its own and runs
to the end of the packet. The packet therefore has one layout rather than two,
and the flag decides what the bytes mean: with `CUSTOM_COLOR` set the colour is
bound to the id, and with it clear the bytes are ignored and any colour left on
the id is cleared — a server that is not overriding the colour sends `0, 0, 0`
and has said nothing. A spawn is a clean slate for both properties, which is what
lets this sub-packet serve as the only spawn packet a server sends to a client
that supports the extension.

## Sub ID 1: Set Flags

Sets the flags of one or more player ids.

| Field Name    | Field Type      | Example | Notes                                     |
|---------------|-----------------|---------|-------------------------------------------|
| Packet ID     | UByte           | `67`    | Always `67`.                              |
| Sub Packet ID | UByte           | `1`     | Always `1` for this sub-packet.           |
| Entries       | SetFlagsEntry[] |         | At least one, see below.                  |

**SetFlagsEntry** (2 bytes)

| Field Name | Field Type | Example  | Notes                          |
|------------|------------|----------|--------------------------------|
| Player ID  | UByte      | `254`    | The player the flags apply to. |
| Flags      | UByte      | `0b1011` | See [Flags](#flags).           |

The entries occupy the remaining bytes of the packet; their count is implied by
the packet length, exactly as for the [ExtInfo](extension.md#extinfo-packet) entries. A
packet with no entry carries no information and should not be sent; a client that
receives one ignores it. If the same id appears twice, the last entry wins.
Servers should keep a packet within the 255 byte budget the base protocol works
with (126 entries) and split larger updates over several packets.

The flags of an id **must** reach the client before the
[Existing Player](../protocol075.md#existing-player) packet that introduces it, on
the same reliable ordered channel. A client that learns about the player first has
already emitted the join notification by the time the flags arrive, and cannot
take it back. This is why the sub-packet carries a list: when a client joins, one
Set Flags packet naming every silent id precedes the whole Existing Player flood,
so the extension adds a single packet to the join sequence no matter how many
silent players the server runs.

Sent during the game, the packet simply changes how an already known player is
presented, with immediate effect and no ordering constraint.

## Sub ID 2: Set Colour

Sets the colour of one or more player ids, as [Set Flags](#sub-id-1-set-flags)
sets their flags.

| Field Name    | Field Type       | Example | Notes                                     |
|---------------|------------------|---------|-------------------------------------------|
| Packet ID     | UByte            | `67`    | Always `67`.                              |
| Sub Packet ID | UByte            | `2`     | Always `2` for this sub-packet.           |
| Entries       | SetColourEntry[] |         | At least one, see below.                  |

**SetColourEntry** (4 bytes)

| Field Name | Field Type | Example | Notes                           |
|------------|------------|---------|---------------------------------|
| Player ID  | UByte      | `254`   | The player the colour applies to. |
| Blue       | UByte      | `160`   | See [Colour](#colour).          |
| Green      | UByte      | `32`    | See [Colour](#colour).          |
| Red        | UByte      | `200`   | See [Colour](#colour).          |

The entries fill the rest of the packet exactly as Set Flags' do, the last entry
for an id wins, and a packet holds at most 63 of them inside the same 255 byte
budget. Setting a colour does not set `CUSTOM_COLOR`: the two travel separately
and a client applies whichever it has, so a colour sent to an id whose flag is
clear is simply held until the flag is set, and the flag alone leaves the player
team-coloured.

A colour is its own sub-packet rather than three more bytes on a Set Flags entry
because that entry is two bytes and its count is read from the packet length. A
mask change is also far commoner than a colour change — colours are set at spawn
and rarely again — so the two packets keep each other small.

Set Colour has no ordering constraint of its own: unlike a mask, a colour that
arrives late costs a frame drawn in the team colour and nothing that cannot be
taken back. A server catching a joining client up still sends it alongside the
Set Flags packet, before the [Existing Player](../protocol075.md#existing-player)
flood, so the client's first frame is right.

### Lifetime

Flags and colour are bound to the **player id**, not to the player occupying it.
They apply until one of:

* a Set Flags, Set Colour or Extended Create Player packet for the same id
  replaces them — a plain [Create Player](../protocol075.md#create-player) does
  not, and each of the three replaces only what it carries;
* the client receives [Player Left](../protocol075.md#player-left) for that id —
  the flags apply to that packet first, so a silent player leaves silently, and
  the id is then reset to a mask of `0` and no colour;
* the world is replaced ([Map Start](../protocol075.md#map-start-075)), which
  resets every id the same way.

Defaulting to `0` means a missed or late Set Flags degrades into an ordinary,
visible player rather than into an invisible one, and a missed Set Colour into a
team-coloured one rather than a black one.

### Notes

Killing a silent player is a normal [Kill Action](../protocol075.md#kill-action)
and removing one a normal [Player Left](../protocol075.md#player-left); the flags
are what keep both quiet. Ids above `31` require the
[Player Limit](player-limit.md) extension, so silent ids are best allocated
downwards from `254`, leaving the low ids to human players.

The hiding flags change what is reported, never what is drawn: a silent player is
rendered, heard, hit and answered in chat like any other, and `CUSTOM_COLOR`
changes only which colour that rendering uses. `HIDE_PRESENCE` and
`HIDE_KILLFEED` suppress third-party notifications only — the receiving player is
still told about their own death or their own kill, silent player named as usual.
Objective announcements ([Intel Capture](../protocol075.md#intel-capture) and the
like) carry their own packets and are out of scope.

Silent players are not participants for the server browser either: they are left
out of the master server [Count Update](../protocolmaster.md#count-update) and of
the advertised capacity, so only human players are counted.

Clients that did not negotiate the extension never receive any of these
sub-packets;
what they are served instead — an ordinary [Create Player](../protocol075.md#create-player),
nothing at all, or a disconnect — is entirely the server's decision.

See [Extensions](extension.md) for how the extension is negotiated.
