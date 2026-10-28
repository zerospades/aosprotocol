# Command Completion

The server tells the client which chat commands it accepts and what each one
takes, so that a client can complete them as they are typed, show their help,
and ask for a missing argument.

Commands stay `/` [Chat Messages](../protocol075.md#chat-message). The extension
only describes them, and the server parses and checks them as before.

| ------------: | ------------- |
| Extension ID: | `0x38`        |
| Packet ID:    | `0x78`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name     | Direction        | Size |
|--------|----------|------------------|------|
| 0      | Commands | Server -> Client | 2+   |
| 1      | Remove   | Server -> Client | 2+   |
| 2      | Clear    | Server -> Client | 2    |

Server to client only; a server that receives one drops it. The client's
commands start empty, and these three change them.

## Sub ID 0: Commands

Adds each command listed. A command replaces every command it shares a name
with, so a server changes a command by sending it again.
Commands not touched are left as they are.

| Field Name    | Field Type | Example | Notes                                 |
|---------------|------------|---------|---------------------------------------|
| Packet ID     | UByte      | `0x78`  | Always `0x78`.                        |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.       |
| Commands      | Command[]  |         | The remaining bytes, see [Command](#command). |

Names, Help, param Names and Choices are of type **String**: NUL-terminated, as
elsewhere in the protocol, and encoded the way a
[Chat Message](../protocol075.md#chat-message) is: Code Page 437, or UTF-8
behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). No String has a length
limit of its own.

### Command

| Field Name  | Field Type | Example              | Notes                                 |
|-------------|------------|----------------------|---------------------------------------|
| Name Count  | UByte      | `2`                  | At least `1`.                         |
| Names       | String[]   | `"teleport"`, `"tp"` | Without the `/`, see [Names](#names). |
| Help        | String     | `"Moves a player"`   | Drawn as chat is drawn.               |
| Param Count | UByte      | `2`                  |                                       |
| Params      | Param[]    |                      | See [Param](#param).                  |

Every name works alike. A client lists the command once, by its first name.

A client looks commands up by name, and completes them by prefix. In a
language with no built-in map, a [crit-bit tree](https://cr.yp.to/critbit.html)
does both, more lightly than a hash table. The order it shows them in is its
own.

A server lists only the commands this player may run now, and adds or removes
some when that changes, say after a login.

### Param

| Field Name   | Field Type | Example    | Notes                                     |
|--------------|------------|------------|-------------------------------------------|
| Type         | UByte      | `1`        | See [Types](#types).                      |
| Flags        | UByte      | `0b000`    | See below.                                |
| Name         | String     | `"player"` | What a prompt or a hint calls it.         |
| Choice Count | UByte      | `3`        | Only when Type is `CHOICE`.               |
| Choices      | String[]   | `"blue"`   | Only when Type is `CHOICE`, see [Choices](#choices). |

| Bit | Name       | Meaning                                                         |
|-----|------------|-----------------------------------------------------------------|
| 0   | `OPTIONAL` | The command runs without it. Every param after it is optional too. |
| 1   | `SECRET`   | A password or the like: masked while typed, kept out of history. |
| 2-7 | reserved   | Must be `0`. Clients **must** ignore unknown bits.              |

Arguments are positional.

### Types

| Value     | Name      | Arguments | The client sends                              | Token |
|-----------|-----------|-----------|-----------------------------------------------|-------|
| `0`       | `TEXT`    | 1         | Any text.                                     |       |
| `1`       | `PLAYER`  | 1         | `#` and the player id in decimal, as in `#7`. | `%p`  |
| `2`       | `CHOICE`  | 1         | One of its [Choices](#choices), spelled as listed. |  |
| `3`       | `BLOCK`   | 3         | A block: `x y z`, as the Pie Menu's `%b`.     | `%b`  |
| `4`       | `POINT`   | 3         | A point: `x y z`, as the Pie Menu's `%h`.     | `%h`  |
| `5`       | `RAY`     | 6         | A ray: `x y z dx dy dz`, as the Pie Menu's `%r`. | `%r` |
| `6`-`255` | reserved  |           | A client treats an unknown Type as `TEXT`.    |       |

A param takes as many space-separated arguments as its row says. Its Token is
what a client offers to complete it with, see [Aim tokens](#aim-tokens).

There is no number type: the arguments servers take that are not words are
durations such as `30min` or `perma`, which no number type fits. A toggle is a
`CHOICE` of `on` and `off`.

A client's player picker inserts `#id`, since names repeat and hold spaces. A
name the player types goes out as typed: piqueserver and LSD also accept part of
a name.

### Choices

A `CHOICE` param takes one value from a fixed set the server lists with it: a
team, a mode, `on` or `off`. The client offers exactly those, as completions or
a picker.

So `/switch` with params `PLAYER player` and `CHOICE team` listing `blue`,
`green` and `spectator` lets a client offer those three, and sends
`/switch #7 green`. Like every type, the set guides the player; it does not
stop them typing something else.

### Names

A command's names hold no whitespace and do not start with `/`. A client matches
them ignoring ASCII case only: `A`-`Z` match `a`-`z`, and every other byte
matches only itself. A Choice is any String without `"` or `\`, which servers
read differently.

## Sub ID 1: Remove

Removes each command named, by any of its names. A name the client does not
have is ignored.

| Field Name    | Field Type | Example    | Notes                           |
|---------------|------------|------------|---------------------------------|
| Packet ID     | UByte      | `0x78`     | Always `0x78`.                  |
| Sub Packet ID | UByte      | `1`        | Always `1` for this sub-packet. |
| Names         | String[]   | `"kick"`   | The remaining bytes, without the `/`. |

## Sub ID 2: Clear

Removes every command, for example after a logout.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x78`  | Always `0x78`.                  |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet. |

## Aim tokens

A player may type a [Pie Menu](pie-menu.md#substitutions) token as an argument,
and the client replaces it with what they aim at, as a slice would:

```
/votekick %p        sends  /votekick #7
/pingmark %h Enemy  sends  /pingmark 256.37 298.02 40.00 Enemy
```

The tokens are `%p`, `%b`, `%h` and `%r`, meaning and written exactly as the
Pie Menu defines them, from the aim when the line is sent. A client offers a
param's [Token](#types) among its completions, so `/votekick ` offers `%p`
beside the player picker.

A token is replaced only when it is a whole word outside double quotes, in a
line whose first word is a listed command's name. Anything else goes out as
typed, `100%`, `%%` and `"%p"` included, so quoting a token sends it literally.
When a token would be empty because nothing is aimed at, the client does not
send the line and tells the player why.

## The line a client sends

The line the player typed, with its [aim tokens](#aim-tokens) replaced. Where
the client inserts an argument itself, a completion or a pick, it separates it
by a single space and wraps it in double quotes if it holds whitespace:

```
/kick #7 "spawn camping"
```

That is the only quoting LSD and piqueserver read alike. A client does not
quote, escape or re-space what the player typed.

The commands are advisory. A client **must not** refuse to send a command it was
not told about, or an argument that does not fit its type, since the server may
accept commands it does not list. A client that has no commands behaves as it would
without the extension.

## Language

Help and param Names **should** be in the player's User locale, from a
[Version Response (Extended)](../protocol075.md#version-response-extended-ep).
Command names and Choices are never translated. Commands sent before the locale
arrives are sent again after it.

## Connection state

The commands belong to the connection, not to the world.
[Map Start](../protocol075.md#map-start-075) does not clear them. A server whose
commands change with the map updates them.

See [Extensions](extension.md) for how the extension is negotiated.
