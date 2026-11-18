# Client Key

Gives a client an identity that servers can recognise, and an encrypted channel
for chat and commands. The client proves it holds a private key, and servers
remember the matching public key: to log a player in again without asking, to
recognise the same player on several servers, for example for a shared
scoreboard, or to give an admin their rights.

| ------------: | ------------- |
| Extension ID: | 2             |
| Packet ID:    | 66            |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name        | Direction         | Size   |
|--------|-------------|-------------------|--------|
| 0      | Challenge   | Server -> Client  | 66     |
| 1      | Response    | Client -> Server  | 136    |
| 2      | Secure Chat | Client <-> Server | 22 + n |

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

A client lets the player see and copy their public key. How it makes and stores
its private key is up to it; a key that grants admin rights is best kept in the
system's key store or behind a passphrase. A client should let the player turn
the extension off. A client with it turned off does not list it in its ExtInfo.

## Admin Keys

A server may learn a player's key however it likes, for convenience such as
logging them in again. An admin key is one its operator added by hand, copied
from the player outside the game, so its login never crosses the network.

Which commands are privileged is the server's choice. It accepts the privileged
commands of an admin key only through [Secure Chat](#sub-id-2-secure-chat); the
same commands in a plain [Chat Message](../protocol075.md#chat-message) are
treated as any player's. Admins who log in another way, such as a password, are
unaffected.

## Sub ID 0: Challenge

| Field Name     | Field Type | Example | Notes                                 |
|----------------|------------|---------|---------------------------------------|
| Packet ID      | UByte      | `66`    | Always `66`.                          |
| Sub Packet ID  | UByte      | `0`     | Always `0` for this sub-packet.       |
| Nonce          | UByte[32]  |         | Random bytes.                         |
| Server Session | UByte[32]  |         | The server's X25519 key for this connection. |

The server sends it at most once per connection, after the client's
[ExtInfo](extension.md#extinfo-packet) lists the extension. The nonce and the
Server Session key come from a cryptographically secure random source and are
new for every connection.

ExtInfo, and so the Challenge, may come after the player has joined, so the
server cannot rely on the key before it has verified a Response. A client
ignores any Challenge after the first.

## Sub ID 1: Response

| Field Name     | Field Type | Example        | Notes                                        |
|----------------|------------|----------------|----------------------------------------------|
| Packet ID      | UByte      | `66`           | Always `66`.                                 |
| Sub Packet ID  | UByte      | `1`            | Always `1` for this sub-packet.              |
| Public Key     | UByte[32]  |                | The client's key.                            |
| Address        | UByte[4]   | `7F 00 00 01`  | The IPv4 address connected to.               |
| Port           | LE Ushort  | `77 80`        | The port connected to.                       |
| Client Session | UByte[32]  |                | The client's X25519 key for this connection. |
| Signature      | UByte[64]  |                | See below.                                   |

The address is the one the client sends its packets to, after resolving any
host name, with the four numbers of `a.b.c.d` in the order `a`, `b`, `c`,
`d`: `127.0.0.1` is `7F 00 00 01`. The port `32887` is `77 80`. The Client
Session key is random and new for every connection.

The signature is over these 118 bytes:

| Field Name     | Field Type | Notes                                                  |
|----------------|------------|--------------------------------------------------------|
| Prefix         | UByte[16]  | The ASCII text `AoS client key 1`, with no terminator. |
| Nonce          | UByte[32]  | From the Challenge.                                    |
| Address        | UByte[4]   | As in the Response.                                    |
| Port           | LE Ushort  | As in the Response.                                    |
| Server Session | UByte[32]  | From the Challenge.                                    |
| Client Session | UByte[32]  | As in the Response.                                    |

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
connected to, which the other server rejects. The two session keys are signed
too, so nobody on the path can swap in their own.

## Session Keys

Once the server has verified a Response, both sides compute the X25519 shared
secret of the two session keys ([RFC 7748](https://www.rfc-editor.org/rfc/rfc7748)),
and from it two 32-byte keys with HKDF-SHA256
([RFC 5869](https://www.rfc-editor.org/rfc/rfc5869)), the nonce as salt:

| Key             | HKDF info                                   |
|-----------------|---------------------------------------------|
| Client to server | `AoS client key 1` followed by ASCII `c2s` |
| Server to client | `AoS client key 1` followed by ASCII `s2c` |

Either side that gets an all-zero shared secret stops: the server does not
recognise the client, and the client sends no Secure Chat. Both sides forget the
keys when the connection ends.

## Sub ID 2: Secure Chat

A [Chat Message](../protocol075.md#chat-message), encrypted.

Once the server has verified a Response, it sends the client a Secure Chat with
counter `0`, such as a system message saying they are recognised. The client
sends Secure Chat only after it has received that one; until then, or if it
never comes, the client is not recognised.

| Field Name    | Field Type | Example | Notes                                     |
|---------------|------------|---------|-------------------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                              |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet.           |
| Counter       | LE Uint    | `0`     | See below.                                |
| Message       | Byte[]     |         | The encrypted message, `n` bytes.         |
| Tag           | UByte[16]  |         | The authentication tag, ending the packet. |

The plaintext is a Chat Message after its Packet ID: player id, chat type and
text, ending with the NUL that clients and servers send. The receiver handles it
as that Chat Message; the server takes the player id to be the sender's, as it
does for any chat. The server sends Secure Chat only for what is meant for that
player alone, such as a command's answer or a private message.

It is encrypted with ChaCha20-Poly1305
([RFC 8439](https://www.rfc-editor.org/rfc/rfc8439)), with the key of its
direction, the counter followed by eight zero bytes as the 12-byte nonce, and
the Packet ID, Sub Packet ID and Counter as additional data. A side whose counter
would wrap past `4294967295` stops sending Secure Chat.

The counter starts at `0` in each direction and goes up by one with every
message. The receiver drops a message whose counter is not above the last one it
accepted, or whose tag does not verify.

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
such as `/logout`. The server grants whatever rights its operator gave that key.

A server may weigh other signs before trusting a key, such as where the
player's IP address is located, a change of address or network since their last
connection, or a long absence. When something looks wrong, it ignores the key
for that connection and asks the player to log in again.

The rest of the connection is not encrypted, so someone who can read and inject
packets can still play as a recognised player, but cannot read or forge Secure
Chat. The server itself is not authenticated: someone who poses as the server
can read what the player sends it, but cannot log in elsewhere as them or pass
their commands on.

See [Extensions](extension.md) for how the extension is negotiated.
