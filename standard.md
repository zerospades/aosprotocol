# Standard

The small rules every extension follows unless its own page says otherwise. They
come from the base protocol and from what reviewers have asked of extensions so
far. A new extension that keeps to them needs to explain only what is its own.

## Numbers

* Little-endian. Signed integers are two's complement, floats are IEEE 754
  single precision, and fields follow each other with no padding.
* Colours are three bytes in the order **blue, green, red**, as in
  [Set Colour](protocol075.md#set-colour).
* Extension ids and packet ids are written in hex, as `0x34` and `0x74`. The
  packet id is `0x40` plus the extension id, see
  [Extension IDs](extensions/extension.md#extension-ids).

## Strings

* **NUL-terminated**, as everywhere in the base protocol. No length prefix.
* **Code Page 437, or UTF-8 behind a `0xff` prefix**, as a
  [Chat Message](protocol075.md#chat-message) is, see
  [UTF-8 Chat](extensions/utf-8-chat.md). A reader handles both. A writer uses
  CP437 when every character has a CP437 byte, and `0xff` and UTF-8 otherwise.
* Text behind the `0xff` prefix is well-formed UTF-8, or the string is invalid.
* **Limits are in bytes**, counting a `0xff` prefix but not the NUL.
* An empty string is valid unless the field says otherwise.

The Reason of a [Teamplay Ping](extensions/teamplay.md#sub-id-1-ping) predates
these rules: it is plain UTF-8 to the end of the packet.

## Players and teams

* Player id `255` is the server, never a player, see
  [Player Limit](extensions/player-limit.md).
* In a chat command, a player is written `#` and their id, as `#7`. Names repeat
  and hold spaces.
* Teams are `0` and `1`, and `255` for spectators. `2` is neutral, for objects
  that no team owns, and is never a player's team.

## Sub-packets

* Every extension packet is its packet id, then a sub-packet id, then its data,
  see [Extension Packets](extensions/extension.md#extension-packets).
* An extension page lists its sub-packets in a table of Sub ID, Name, Direction
  and Size, then gives each one a section headed `Sub ID n: Name`.
* A side that receives a sub-packet meant for the other direction drops it.
* A list is preceded by its count, so that one wrong length cannot swallow the
  record after it and still parse.

## Room to grow

* **Reserved bits** are sent as `0` and ignored on receipt.
* **Reserved bytes**, such as a Message ID kept for a later version, are sent as
  `0`, and a non-zero value is ignored.
* **An unknown value** in an enum is read as a fallback the extension names,
  such as "does nothing", rather than refusing the packet.
* A flag never changes the layout of a packet: a receiver could not ignore it.
  A new field means a new version.

## Validation

* A receiver parses a whole packet before applying any of it, and applies it
  whole or not at all.
* A refused packet leaves the previous state as it was. The receiver logs it and
  does not disconnect.
* Whatever reaches the server is checked by the server. A client's packet is a
  request, never a fact.

## State

* Extensions are negotiated once per connection, see
  [Negotiation Flow](extensions/extension.md#negotiation-flow).
* Each extension says what [Map Start](protocol075.md#map-start-075) and
  [Player Left](protocol075.md#player-left) clear. Map Start replaces the world,
  not the connection.

## Chat and commands

* Anything an extension has the client send as chat goes out the way the client
  normally sends chat. An empty chat line is valid.
* Rate-limiting chat is the server's business.
* Gameplay a client only needs to trigger is better as a
  [command](commands.md) than as a new packet.

## Language

* Text a player reads **should** be in their User locale, from a
  [Version Response (Extended)](protocol075.md#version-response-extended-ep). A
  server matches the language and region, then the language alone, then falls
  back to its own default, which it also uses with no locale.
* Text the server parses, such as a command, a name or a choice, is never
  translated.
