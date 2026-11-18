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
| 1      | Response  | Client -> Server | 98   |

## Keys

Keys are Ed25519 ([RFC 8032](https://www.rfc-editor.org/rfc/rfc8032)).

A client uses the same key for every server, so every server sees the same
public key. A client may instead use a different key for some or all servers;
those servers then cannot tell it is the same player, and nothing shared
between servers, such as a scoreboard, follows it.

How the client makes and stores its key is up to it. A client should let the
player turn the extension off. A client with it turned off does not answer a
Challenge.

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

| Field Name    | Field Type | Example | Notes                             |
|---------------|------------|---------|-----------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                      |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.   |
| Public Key    | UByte[32]  |         | The client's key for this server. |
| Signature     | UByte[64]  |         | See below.                        |

The signature is over these 54 bytes:

| Bytes | Content                                                       |
|-------|---------------------------------------------------------------|
| 16    | The ASCII text `AoS client key 1`, with no terminator.        |
| 32    | The nonce.                                                    |
| 4     | The IPv4 address the client connected to, in network order.  |
| 2     | The port the client connected to, LE.                         |

The address is the one the client sent its packets to, after resolving any host
name. The prefix keeps the signature from being valid for anything else.

The client answers a Challenge once, or not at all. The server drops a Response
it did not ask for, a second Response, and a Response whose signature does not
verify for an address the server is reached at; the client is then simply not
recognised. A server behind NAT or a proxy is told its public address by its
operator.

The address stops a server from passing another server's Challenge on to a
player and logging in there as them: the player signs the address they
connected to, which the other server rejects.

## Server Use

What the server does with the key is its own business. The usual flow:

1. The key is linked to an account: the player is logged in to it.
2. The key is unknown: nothing happens. When the player later logs in, with
   whatever the server uses (`/login`, `/register`, ...), the server links the
   key to that account.

Servers may share public keys with each other, to recognise the same player
across servers. A public key lets nobody log in, so a leaked or shared list of
keys is safe.

A key proves the same client install, not the same person: anyone using the
player's computer is recognised too. A server may offer a way to unlink a key,
such as `/logout`, and may refuse to grant admin rights from a key alone.

As with the rest of the protocol the connection is not encrypted, so someone
who can read and inject packets can still take over a connection once it is
recognised. Unlike a password typed in chat, no secret is ever sent.

See [Extensions](extension.md) for how the extension is negotiated.
