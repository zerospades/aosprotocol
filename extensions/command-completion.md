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
| 0      | Commands | Server -> Client | 3+   |
| 1      | Remove   | Server -> Client | 3+   |
| 2      | Clear    | Server -> Client | 2    |

Server to client only; a server that receives one drops it. The client's
commands start empty, and these three change them.

## Sub ID 0: Commands

Adds each command listed, or replaces the one of the same Name. Commands not
listed are left as they are.

| Field Name    | Field Type | Example | Notes                                         |
|---------------|------------|---------|-----------------------------------------------|
| Packet ID     | UByte      | `0x78`  | Always `0x78`.                                |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet.               |
| Command Count | UByte      | `12`    | Commands that follow.                         |
| Commands      | Command[]  |         | See [Command](#command).                      |

Names, Aliases, Help, param Names and Choices are of type **String**:
NUL-terminated, as elsewhere in the protocol, and encoded the way a
[Chat Message](../protocol075.md#chat-message) is: Code Page 437, or UTF-8
behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md). No String has a length
limit of its own.

### Command

| Field Name  | Field Type | Example              | Notes                                 |
|-------------|------------|----------------------|---------------------------------------|
| Name        | String     | `"teleport"`         | Without the `/`, not empty, see [Names](#names). |
| Alias Count | UByte      | `1`                  | Aliases that follow.                  |
| Aliases     | String[]   | `"tp"`               | Other names for the same command.     |
| Help        | String     | `"Moves a player"`   | Drawn as chat is drawn. |
| Param Count | UByte      | `2`                  | Params that follow.                   |
| Params      | Param[]    |                      | See [Param](#param).                  |

Each Alias is a String like the Name. A client completes an alias like the
command, and lists the command once.

A client looks commands up by Name and Alias, and completes them by prefix. In a
language with no built-in map, a [crit-bit tree](https://cr.yp.to/critbit.html)
does both, more lightly than a hash table.

A client shows commands in the order they were first added. A server lists only
the commands this player may run now, and adds or removes some when that
changes, say after a login.

### Param

| Field Name   | Field Type | Example    | Notes                                         |
|--------------|------------|------------|-----------------------------------------------|
| Type         | UByte      | `1`        | See [Types](#types).                          |
| Flags        | UByte      | `0b000`    | See below.                                    |
| Name         | String     | `"player"` | What a prompt or a hint calls it. |
| Choice Count | UByte      | `0`        | `0` unless Type is `CHOICE`.                  |
| Choices      | String[]   | `"blue"`   | See [Choices](#choices).                      |

| Bit | Name       | Meaning                                                         |
|-----|------------|-----------------------------------------------------------------|
| 0   | `OPTIONAL` | The command runs without it. Every param after it is optional too. |
| 1   | `SECRET`   | A password or the like: masked while typed, kept out of history. |
| 2-7 | reserved   | Must be `0`. Clients **must** ignore unknown bits.              |

Arguments are positional.

### Types

| Value     | Name      | The client sends                                       |
|-----------|-----------|--------------------------------------------------------|
| `0`       | `TEXT`    | Any text, quoted if it holds a space.                  |
| `1`       | `PLAYER`  | `#` and the player id in decimal, as in `#7`.          |
| `2`       | `CHOICE`  | One of its [Choices](#choices), spelled as listed.     |
| `3`-`255` | reserved  | A client treats an unknown Type as `TEXT`.             |

There is no number type: the arguments servers take that are not words are
durations such as `30min` or `perma`, which no number type fits. A toggle is a
`CHOICE` of `on` and `off`.

A client's player picker inserts `#id`, since names repeat and hold spaces. A
name the player types goes out as typed: piqueserver and LSD also accept part of
a name.

### Choices

A `CHOICE` param takes one value from a fixed set the server lists with it: a
team, a mode, `on` or `off`. The client offers exactly those, as completions or
a picker, and sends the one picked as it is spelled here.

Each Choice is a String, see [Names](#names).

So `/switch` with params `PLAYER player` and `CHOICE team` listing `blue`,
`green` and `spectator` lets a client offer those three, and sends
`/switch #7 green`. Like every type, the set guides the player; it does not
stop them typing something else.

### Names

A command Name or Alias holds no space and does not start with `/`; servers
match it whatever its case. A Choice is any String, and is quoted when sent if it
holds a space.

## Sub ID 1: Remove

Removes the commands named, each by its Name. A name the client does not have is
ignored.

| Field Name    | Field Type | Example    | Notes                           |
|---------------|------------|------------|---------------------------------|
| Packet ID     | UByte      | `0x78`     | Always `0x78`.                  |
| Sub Packet ID | UByte      | `1`        | Always `1` for this sub-packet. |
| Name Count    | UByte      | `1`        | Names that follow.              |
| Names         | String[]   | `"kick"`   | Command Names, without the `/`. |

## Sub ID 2: Clear

Removes every command, for example after a logout.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x78`  | Always `0x78`.                  |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet. |

## Language

Help and param Names **should** be in the player's User locale, from a
[Version Response (Extended)](../protocol075.md#version-response-extended-ep).
Command Names, Aliases and Choices are never translated. Commands sent before
the locale arrives are sent again after it.

## The line a client sends

`/`, the Name, then each argument, separated by single spaces. An argument that
holds a space is wrapped in double quotes, which piqueserver and LSD both read as
one argument:

```
/kick #7 "spawn camping"
```

for a command `kick` with params `PLAYER player` and `TEXT reason`. It goes out
however the client normally sends chat.

The commands are advisory. A client **must not** refuse to send a command it was
not told about, or an argument that does not fit its type, since the server may
accept commands it does not list. A client that has no commands behaves as it would
without the extension.

## Validation

A client parses the whole packet before applying any of it. It refuses the
packet only when a record or a string runs past the end of the packet, or bytes
remain after the last one; its commands then stay as they were, and it logs the
packet rather than disconnecting.

Otherwise a bad command is skipped and the others are applied. A command is bad
when its Name is empty, a String behind the `0xff` prefix is not well-formed
UTF-8, or a `CHOICE` param has no choices. A Name or Alias that another command
already has belongs to the one added last.

## Connection state

The commands belong to the connection, not to the world.
[Map Start](../protocol075.md#map-start-075) does not clear them. A server whose
commands change with the map updates them.

See [Extensions](extension.md) for how the extension is negotiated.
