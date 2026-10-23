# Pie Menu

A pie menu is a ring that opens around the crosshair while a key is held. It is
divided into wedges, and a player picks one by flicking the aim towards it and
releasing the key. Each wedge sends one thing: a chat line, a ping on the world,
or a command for the server to read.

One ring is a **pie** and one wedge is a **slice**. The point of the extension is
that the server chooses what is on offer, since it knows its mode, its commands
and its vocabulary.

| ------------: | ------------- |
| Extension ID: | `0x31`        |
| Packet ID:    | `0x71`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name | Direction        | Size |
|--------|------|------------------|------|
| 0      | Menu | Server -> Client | 3+   |

Server to client only; a server that receives one drops it.

## Sub ID 0: Menu

| Field Name    | Field Type | Example | Notes                                 |
|---------------|------------|---------|---------------------------------------|
| Packet ID     | UByte      | `0x71`  | Always `0x71`.                        |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.       |
| Pie Count     | UByte      | `4`     | Pies that follow.                     |
| Pies          | Pie[]      |         | The remaining bytes, see [Pie](#pie). |

Theme, Text, Label and Tooltip are of type **String**: NUL-terminated, as
elsewhere in the protocol, and encoded the way a
[Chat Message](../protocol075.md#chat-message) does: Code Page 437, or UTF-8
behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). A client draws them as
it draws chat, colour codes included. No string has a length limit of its own;
a client may cut what it draws, and the server treats a line that is too long as
it treats any over-long chat.

### Pie

One ring, captioned in the middle by its theme.

| Field Name       | Field Type | Example    | Notes                                   |
|------------------|------------|------------|-----------------------------------------|
| Contexts         | UByte      | `0b011`    | Where it is offered, see [Contexts](#contexts). |
| Theme Message ID | UByte      | `0`        | Reserved. Must be `0`; a client ignores it. |
| Theme            | String     | `"Social"` | The centre caption. Empty means none. |
| Slices           | Slice[6]   |            | Exactly six, see [Slice](#slice).       |

There is no slice count: a server wanting fewer fills the spare slices with
[`NONE`](#actions), and one wanting more uses another pie.

Pies are offered in the order sent, within each context, and the first opens
first. A client may instead reopen the pie last used in that context.

### Contexts

The [player aimed at](#substitutions), or none, decides which pies are offered.
A bitmask, so a pie that suits more than one context is sent once.

| Bit | Name        | Offered when the menu opens on                     |
|-----|-------------|----------------------------------------------------|
| 0   | `WORLD`     | Terrain, the sky, or nothing in particular.        |
| 1   | `TEAMMATE`  | A player on the opening player's own team.         |
| 2   | `ENEMY`     | A player on the other team.                        |
| 3   | `SPECTATOR` | Anything, while the opening player is a spectator. |
| 4   | `DEAD`      | Anything, while the opening player is dead.        |
| 5-7 | reserved    | Must be `0`. Clients **must** ignore unknown bits. |

A spectator is offered `SPECTATOR` pies alone, and a player who is dead and
waiting to respawn `DEAD` pies alone, such as a choice of where to spawn. A staff member spectating from a
team counts as a spectator. A context with no pies has no
menu, and no other context's pies stand in.

### Slice

| Field Name   | Field Type | Example   | Notes                                      |
|--------------|------------|-----------|--------------------------------------------|
| Action       | UByte      | `1`       | See [Actions](#actions).                   |
| Message ID   | UByte      | `0`       | Reserved. Must be `0`; a client ignores it. |
| Flags        | UByte      | `0b0`     | See below.                                 |
| Text         | String     | `"/apoc"` | What it sends.                             |
| Label        | String     | `"Nuke"`  | What it draws. Empty means draw the Text. |
| Tooltip      | String     | `"Calls in an airstrike"` | Empty means none. |

A client shows the Tooltip while the slice is highlighted and never sends it.
Where and how it is drawn is the client's business, and a client may leave
tooltips out.

| Bit | Name     | Meaning                                                      |
|-----|----------|--------------------------------------------------------------|
| 0   | `GLOBAL` | Goes to the whole server rather than to the team.             |
| 1-7 | reserved | Must be `0`. Clients **must** ignore unknown bits.            |

`GLOBAL` is the team/global bit of the base
[Chat Message](../protocol075.md#chat-message). **There is no enemy channel**, so
the [`ENEMY`](#contexts) context decides which pies are offered, never where
their words go.

### Actions

| Value     | Name      | Choosing the slice sends                                   |
|-----------|-----------|------------------------------------------------------------|
| `0`       | `NONE`    | Nothing. The slice cannot be chosen.                       |
| `1`       | `CHAT`    | Text, as a [Chat Message](../protocol075.md#chat-message). |
| `2`       | `PING`    | A [Teamplay Ping](teamplay.md#sub-id-1-ping), Text as its Reason. |
| `3`       | `COMMAND` | Text, as a [Chat Message](../protocol075.md#chat-message), meant as a command. |
| `4`-`255` | reserved  | A client treats an unknown Action as `NONE`.               |

The Action is also what a client styles a slice by. Whatever a slice sends goes
out as the client normally sends chat. A server should keep control bytes out of
a Text: a client may re-encode it, and `0x01`-`0x1f` mean symbols in Code Page
437 but colours in UTF-8.

**`PING`** marks the point the [aim](#substitutions) met, with the Text decoded
to UTF-8 as its Reason, which may be empty for a neutral marker. It needs
[Teamplay](teamplay.md) negotiated with its `PING` bit set, a living player, and
an aim that met a block or a player. Failing any of these, the client sends the
Text as chat instead, so only the marker is lost.

**`COMMAND`** is sent exactly as `CHAT` is, `GLOBAL` included; servers find a
command by its leading `/` on either channel. It exists so that a client can
style a command apart from a line of chat. The client interprets none of it.

**`NONE`** cannot be chosen: aiming at it selects nothing, as the centre does. It
may still carry a Label to show something, such as `Airstrikes: 2` beside the
slice that calls one. How it looks is the client's business, but a client must
not make it choosable.

### Substitutions

The Text of a slice may name what the menu was opened on.

| Token | Expands to          | Example                                     | Type                                                                                     | Empty when             |
|-------|---------------------|---------------------------------------------|------------------------------------------------------------------------------------------|------------------------|
| `%p`  | The player aimed at | `#7`                                        | `#` then an integer, the player id                                                      | No player is aimed at. |
| `%b`  | The block aimed at  | `257 298 40`                                | 3 integers: `x y z`                                                                      | No block is aimed at.  |
| `%r`  | The camera ray      | `255.50 300.12 40.25 0.7071 -0.7071 0.0000` | 6 decimal numbers: the position `x y z` with at least 2 decimal places, then the direction `dx dy dz` with at least 4 | Never.      |
| `%%`  | A percent sign      | `%`                                         |                                                                                          | Never.                 |

A `%` followed by anything else, or by nothing, is reserved, and version 1 leaves
it as it is. An empty token leaves its argument missing, so `/votekick %p` with
no player aimed at sends `/votekick `, not `/votekick #`.

**The aim** is a ray from the camera being drawn, along its view direction:
the player's own view while alive, and whatever view the client shows while dead
or spectating, a free camera or a followed player. It has no length limit; a server that wants one applies it
to what it receives.

* **The player aimed at**, `%p`, is the first player the aim meets before any
  block.
* **The block aimed at**, `%b`, is the first solid block the aim meets. Players do
  not hide it.
* **The camera ray**, `%r`, is the aim itself: the camera position, then the view
  direction, a vector of length `1` in the same
  [coordinates](../protocol075.md#about-coordinates), so `dz` above `0` looks
  down. A server normalises it again before use.

**Types.** The values of a token are separated by single spaces.

* An **integer** is written in base 10, with no sign, leading zeros or padding.
* A **decimal** number has at least the decimal places its row gives, `.` as
  the decimal point whatever the locale, and no exponent.

`%b` suits a simple command. `%r` lets the server cast the ray against its own
map: a spectator edit mode, building and digging from the free camera with
`/place %r`, since a server never otherwise learns where a spectator looks; or
accuracy when the client's map has drifted from the server's.

Only the Text is expanded; Theme, Label and Tooltip are shown as sent. The
client expands once, immediately before sending, and never re-scans the result,
so a name that looks like a token is not one.

### Language

Themes, Labels and Tooltips **should** be in the player's User locale, from a
[Version Response (Extended)](../protocol075.md#version-response-extended-ep). A
Menu sent before the locale arrives is sent again after it.

## Opening the menu

Everything is taken when the menu opens: the context, the aim, and the menu
itself. None of it changes while the menu is held.

## Replacing the client's menu

* **No Menu has arrived.** The client keeps its own.
* **A Menu with pies.** It replaces the client's own and any earlier Menu
  entirely, from the next time the menu is opened. Menus are never merged.
* **A Menu with a Pie Count of `0`.** There is no pie menu here, and the client's
  own menu does not come back.

A client **must not** add slices, reorder them, or drop one it dislikes. It may
refuse the menu outright.

## Geometry

* Slice `0` is at the top, the rest **clockwise**.
* The six slices divide the ring evenly, `60` degrees each, so slice `0` is
  centred on straight up and slice `3` on straight down.
* The centre selects nothing, so releasing the key without choosing is always
  possible.

Everything else — radius, colour, animation, how pies are cycled, hold or toggle
— is the client's and is not on the wire.

## Validation

A client parses the whole packet before applying any of it, and **a menu is
applied whole or not at all**. Where one is refused the previous menu stands and
the client logs it rather than disconnecting. A Menu is refused when:

* A record or a string runs past the end of the packet, or bytes remain after
  the last pie.
* A string behind the `0xff` prefix is not well-formed UTF-8.

## Connection state

The menu belongs to the connection, not to the world and not to a player.
[Map Start](../protocol075.md#map-start-075) and
[Player Left](../protocol075.md#player-left) clear nothing.

See [Extensions](extension.md) for how the extension is negotiated.
