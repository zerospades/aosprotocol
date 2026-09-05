# Silent Player

Lets the server mark player ids as *silent*: players that exist in the world and
are rendered like any other player, but that take no part in the presentation the
client builds around its player list — scoreboard, player counts, presence
notices, kill feed.

The base protocol has a single notion of a player: a client only knows about a
player because it received an [Existing Player](../protocol075.md#existing-player)
or [Create Player](../protocol075.md#create-player) packet for it, and that same
list drives the scoreboard and every player-related notification. This extension
adds a per-player-id presentation mask instead of a second entity concept, so
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

| Sub ID | Name                | Direction        | Size          |
|--------|---------------------|------------------|---------------|
| 0      | Create Silent Player| Server -> Client | `18+`         |
| 1      | Set Flags           | Server -> Client | `2+2*entries` |

Sub packet 0 replaces [Create Player](../protocol075.md#create-player) for a
silent player: it carries the same spawn data plus the flags, so a spawn stays
one packet and cannot be misread by a client that has not been told about the
flags yet. Sub packet 1 covers everything a spawn cannot: the players already in
the world when a client joins, and any change of presentation while the game
runs.

## Flags

Both sub-packets carry the same one byte mask.

| Bit | Name             | Meaning                                                                                                                                              |
|-----|------------------|------------------------------------------------------------------------------------------------------------------------------------------------------|
| 0   | `HIDE_ROSTER`    | The player is left out of the scoreboard, of the total / alive / spectator counts, and of spectator follow-camera cycling and similar player pickers. |
| 1   | `HIDE_PRESENCE`  | No join, team change or leave notification is shown for this player.                                                                                 |
| 2   | `HIDE_KILLFEED`  | No kill feed line is shown for a kill this player made or suffered.                                                                                  |
| 3   | `NO_STATS`       | The player is ignored by client-side statistics: kill and death counters, kill streaks, and domination / revenge tracking.                            |
| 4-7 | reserved         | Must be `0`. Clients must ignore unknown bits.                                                                                                       |

The bits are independent, which is the point of the mask. An RPG server that
wants its mobs to appear and vanish without a word but still wants kills
announced sets `HIDE_ROSTER | HIDE_PRESENCE | NO_STATS` and leaves
`HIDE_KILLFEED` clear; a fully silent actor sets all four.

Presence is a single bit rather than one for joining and one for leaving: a
client that announced neither arrival nor departure is coherent, and one that
announces only the departure of a player it never announced is not.

A mask of `0` means the player is presented normally. That is how a player is
un-silenced — there is no separate clear sub-packet — and it is what a server
sends to reveal an actor that was silent until then, an ambushing NPC turning
into a scoreboard participant for example.

## Sub ID 0: Create Silent Player

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
| Name          | CP437 String | `Wolf`   | As in Create Player, same encoding, not `\0` terminated. |

Everything after Player ID behaves exactly as in Create Player, including the
spawn height adjustment clients apply; the only addition is Flags, which the
client stores for the id and applies before it emits anything about the spawn.
The packet is therefore atomic: there is no window in which the client considers
the player an ordinary participant.

A mask of `0` spawns an ordinary player and clears any flags left on the id, so
this sub-packet can serve as the only spawn packet a server sends to a client
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

### Lifetime

Flags are bound to the **player id**, not to the player occupying it. They apply
until one of:

* a Set Flags or Create Silent Player packet for the same id replaces them —
  a plain [Create Player](../protocol075.md#create-player) does not;
* the client receives [Player Left](../protocol075.md#player-left) for that id —
  the flags apply to that packet first, so a silent player leaves silently, and
  the id is then reset to `0`;
* the world is replaced ([Map Start](../protocol075.md#map-start-075)), which
  resets every id to `0`.

Defaulting to `0` means a missed or late Set Flags degrades into an ordinary,
visible player rather than into an invisible one. Flags for the client's own
player id are ignored: a client is never silent to itself.

### Notes

Killing a silent player is a normal [Kill Action](../protocol075.md#kill-action)
and removing one a normal [Player Left](../protocol075.md#player-left); the flags
are what keep both quiet. Ids above `31` require the
[Player Limit](player-limit.md) extension, so silent ids are best allocated
downwards from `254`, leaving the low ids to human players.

The flags change what is reported, never what is drawn: a silent player is
rendered, heard, hit and answered in chat like any other. `HIDE_PRESENCE` and
`HIDE_KILLFEED` suppress third-party notifications only — the receiving player is
still told about their own death or their own kill, silent player named as usual.
Objective announcements ([Intel Capture](../protocol075.md#intel-capture) and the
like) carry their own packets and are out of scope.

Silent players are not participants for the server browser either: they are left
out of the master server [Count Update](../protocolmaster.md#count-update) and of
the advertised capacity, so only human players are counted.

Clients that did not negotiate the extension never receive either sub-packet;
what they are served instead — an ordinary [Create Player](../protocol075.md#create-player),
nothing at all, or a disconnect — is entirely the server's decision.

See [Extensions](extension.md) for how the extension is negotiated.
