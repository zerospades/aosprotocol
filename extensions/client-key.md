# Client Key

Lets a server recognise a client it has seen before. A player logs in once, with
whatever the server uses (`/login`, `/register`, ...), and the server logs them
in again on later connections without asking. The client proves it holds a
private key, and the server remembers the matching public key.

| ------------: | ------------- |
| Extension ID: | 2             |
| Packet ID:    | 66            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name      | Direction        | Size |
|--------|-----------|------------------|------|
| 0      | Challenge | Server -> Client | 34   |
| 1      | Response  | Client -> Server | 98   |

## Keys

Keys are Ed25519 ([RFC 8032](https://www.rfc-editor.org/rfc/rfc8032)).

A client uses a different key for each server, chosen by the address it
connected to (IP and port), and never shows the same public key to two
servers. One key for all servers would:

* let servers follow a player from one server to another, and
* let a server log in to another server as its players: it connects to the
  other server, passes that server's Challenge on to a player, and sends back
  the player's Response.

With a key per address, the player signs with the key for the server they
connected to, which the other server does not know.

How the client makes and stores its keys is up to it. It can generate a random
key the first time it meets an address, or derive it from a single secret, for
example the first 32 bytes of `HMAC-SHA512(secret, "aos client key" || address)`
as the Ed25519 seed, so there is only one secret to back up.

A client should let the player turn the extension off, and forget the key for a
server. A client with it turned off does not answer a Challenge.

## Sub ID 0: Challenge

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                    |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Nonce         | UByte[32]  |         | Random bytes.                   |

The server sends it at most once per connection, after the client's
[ExtInfo](extension.md#extinfo-packet) lists the extension. The nonce comes
from a cryptographically secure random source and is new for every connection.

## Sub ID 1: Response

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                    |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet. |
| Public Key    | UByte[32]  |         | The client's key for this server. |
| Signature     | UByte[64]  |         | See below.                      |

The signature is over the 16 ASCII bytes `AoS client key 1`, with no
terminator, followed by the 32 bytes of the nonce. The prefix keeps the
signature from being valid for anything else.

The client answers a Challenge once, or not at all. The server drops a Response
it did not ask for, a second Response, and a Response whose signature does not
verify; the client is then simply not recognised.

## Server Use

What the server does with the key is its own business. The usual flow:

1. The signature verifies and the key is linked to an account: the player is
   logged in to it.
2. The key is unknown: nothing happens. When the player later logs in, the
   server links the key to that account.

The server stores only public keys, so a leaked database lets nobody log in.

A key proves the same client install, not the same person: anyone using the
player's computer is logged in too. A server may offer a way to unlink a key,
such as `/logout`, and may refuse to grant admin rights from a key alone.

A key is lost when the client forgets it or the server changes address; the
player logs in again and the server links the new key.

As with the rest of the protocol the connection is not encrypted, so someone
who can read and inject packets can still take over a connection once it is
logged in. Unlike a password typed in chat, no secret is ever sent.

See [Extensions](extension.md) for how the extension is negotiated.
