# Client Key

Gives a client an identity that servers can recognise. The client proves it
holds a private key, and servers remember the matching public key: to log a
player in again without asking, or to recognise the same player on several
servers, for example for a shared scoreboard.

| ------------: | ------------- |
| Extension ID: | 2             |
| Packet ID:    | 66            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name      | Direction        | Size |
|--------|-----------|------------------|------|
| 0      | Challenge | Server -> Client | 34   |
| 1      | Response  | Client -> Server | 104  |

## Keys

Keys are Ed25519 ([RFC 8032](https://www.rfc-editor.org/rfc/rfc8032)), plain,
not Ed25519ph or Ed25519ctx. A server verifies strictly: it rejects a signature
whose S is not below the group order, a key or point that is not canonically
encoded, and a public key of small order.

A client uses the same key for every server. The key is a public, permanent
identifier: every server the client answers learns it, and servers can link the
player across the whole server list. A client may use a different key for some
or all servers; those servers then cannot tell it is the same player, and
nothing shared between servers, such as a scoreboard, follows it.

How the client makes and stores its key is up to it. A client should let the
player turn the extension off. A client with it turned off does not list it in
its ExtInfo.

## Sub ID 0: Challenge

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                    |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Nonce         | UByte[32]  |         | Random bytes.                   |

The server sends it at most once per connection, after the client's
[ExtInfo](extension.md#extinfo-packet) lists the extension. The nonce comes
from a cryptographically secure random source and is new for every connection.

ExtInfo, and so the Challenge, may come after the player has joined, so the
server cannot rely on the key before it has verified a Response. A client
ignores any Challenge after the first.

## Sub ID 1: Response

| Field Name    | Field Type | Example        | Notes                             |
|---------------|------------|----------------|-----------------------------------|
| Packet ID     | UByte      | `66`           | Always `66`.                      |
| Sub Packet ID | UByte      | `1`            | Always `1` for this sub-packet.   |
| Public Key    | UByte[32]  |                | The client's key.                 |
| Address       | UByte[4]   | `7F 00 00 01`  | The IPv4 address connected to.    |
| Port          | LE Ushort  | `77 80`        | The port connected to.            |
| Signature     | UByte[64]  |                | See below.                        |

The address is the one the client sends its packets to, after resolving any
host name, with the four numbers of `a.b.c.d` in the order `a`, `b`, `c`,
`d`: `127.0.0.1` is `7F 00 00 01`. The port `32887` is `77 80`.

The signature is over these 54 bytes:

| Field Name | Field Type | Notes                                                  |
|------------|------------|--------------------------------------------------------|
| Prefix     | UByte[16]  | The ASCII text `AoS client key 1`, with no terminator. |
| Nonce      | UByte[32]  | The nonce from the Challenge.                          |
| Address    | UByte[4]   | As in the Response.                                    |
| Port       | LE Ushort  | As in the Response.                                    |

The prefix keeps the signature from being valid for anything else. IPv4 is the
only address family; another needs a new version of the extension.

The client answers a Challenge once, or not at all. The server drops a Response
it did not ask for, a second Response, and a Response of the wrong size. It then
checks the address and port against its own, and only then the signature. A
Response that fails either check is dropped, and the client is simply not
recognised. The server may tell the player in chat whether they were
recognised.

Its own addresses and ports are those it listens on, the public ones set by its
operator or looked up, and those of any relay its operator runs in front of it.
When a router, container or relay maps the port, the operator sets the port
players connect to. When to look up the public address again is the server's
choice; until it is known, only the other addresses count.

Many unrelated servers share a private address, so a server accepts a loopback
address (`127.0.0.0/8`) only from a connection that comes from loopback, and a
private one (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`,
`169.254.0.0/16`) only from a connection whose source address is in the same
subnet as the server's address that received it.

The address stops a server from passing another server's Challenge on to a
player and logging in there as them: the player signs the address they
connected to, which the other server rejects.

## Server Use

What the server does with a key is its own business; usually it links the key
to an account when the player logs in, and logs them in to it on later
connections. When the same key is on two connections at once, the server
decides which, if any, is recognised.

Servers may share public keys with each other, to recognise the same player
across servers. A list of keys cannot be used to log in, but it still links
players across servers, and shared data is only as trustworthy as the server
that supplied it.

A key proves the same client install, not the same person: anyone using the
player's computer is recognised too. A server may offer a way to unlink a key,
such as `/logout`. A key alone does not grant admin or moderator rights.

As with the rest of the protocol the connection is not encrypted, so someone
who can read and inject packets can still take over a connection once it is
recognised. Unlike a password typed in chat, no secret is ever sent.

See [Extensions](extension.md) for how the extension is negotiated.
