# Client Key

Gives a client an identity that servers recognise, and an encrypted chat for
commands. The client proves it holds a private key; servers remember the public
key.

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

A client uses the same key for every server, so the key is a public, permanent
identifier that links the player across servers. A client may use other keys
for some servers, which then cannot tell it is the same player.

A client lets the player see and copy their public key, and turn the extension
off; it then does not list it in its ExtInfo. How the client stores its private
key is up to it, ideally in the system's key store for an admin key.

## Sub ID 0: Challenge

| Field Name     | Field Type | Example | Notes                                        |
|----------------|------------|---------|----------------------------------------------|
| Packet ID      | UByte      | `66`    | Always `66`.                                 |
| Sub Packet ID  | UByte      | `0`     | Always `0` for this sub-packet.              |
| Nonce          | UByte[32]  |         | Random bytes.                                |
| Server Session | UByte[32]  |         | The server's X25519 key for this connection. |

The server sends it at most once per connection, after the client's
[ExtInfo](extension.md#extinfo-packet) lists the extension, so possibly after
the player has joined; it cannot rely on the key before verifying a Response.
The nonce and the Server Session key are new for every connection, from a
cryptographically secure random source. A client ignores any Challenge after
the first.

## Sub ID 1: Response

| Field Name     | Field Type | Example       | Notes                                        |
|----------------|------------|---------------|----------------------------------------------|
| Packet ID      | UByte      | `66`          | Always `66`.                                 |
| Sub Packet ID  | UByte      | `1`           | Always `1` for this sub-packet.              |
| Public Key     | UByte[32]  |               | The client's key.                            |
| Address        | UByte[4]   | `7F 00 00 01` | The IPv4 address connected to.               |
| Port           | LE Ushort  | `77 80`       | The port connected to.                       |
| Client Session | UByte[32]  |               | The client's X25519 key for this connection. |
| Signature      | UByte[64]  |               | See below.                                   |

The address is the one the client sends its packets to, after resolving any
host name, as `a`, `b`, `c`, `d` for `a.b.c.d`. The Client Session key is new
for every connection.

The signature is over these 118 bytes:

| Field Name     | Field Type | Notes                                                  |
|----------------|------------|--------------------------------------------------------|
| Prefix         | UByte[16]  | The ASCII text `AoS client key 1`, with no terminator. |
| Nonce          | UByte[32]  | From the Challenge.                                    |
| Address        | UByte[4]   | As in the Response.                                    |
| Port           | LE Ushort  | As in the Response.                                    |
| Server Session | UByte[32]  | From the Challenge.                                    |
| Client Session | UByte[32]  | As in the Response.                                    |

IPv4 is the only address family; another needs a new version.

The client answers a Challenge once, or not at all. The server drops a Response
it did not ask for, a second one, one of the wrong size, one whose address and
port are not its own, and, checked last, one whose signature does not verify.
A dropped Response leaves the client unrecognised.

A server's own addresses and ports are those it listens on, its public ones,
set by its operator or looked up, and those of any relay its operator runs.
Until a looked-up address is known, it does not count.
When the port is mapped, the operator sets the one players connect to. It
accepts a loopback address only from a loopback connection, and a private one
(`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `100.64.0.0/10`,
`169.254.0.0/16`) only from a source in the same subnet as the address that
received it.

Signing the address stops a server from relaying another server's Challenge to
log in there as the player. Signing the session keys stops anyone on the path
from swapping in their own.

## Session Keys

Once the server has verified a Response, both sides compute the X25519 shared
secret of the two session keys ([RFC 7748](https://www.rfc-editor.org/rfc/rfc7748)),
and from it one 32-byte key per direction with HKDF-SHA256
([RFC 5869](https://www.rfc-editor.org/rfc/rfc5869)), the nonce as salt:

| Key              | HKDF info                                   |
|------------------|---------------------------------------------|
| Client to server | `AoS client key 1` followed by ASCII `c2s` |
| Server to client | `AoS client key 1` followed by ASCII `s2c` |

A side that gets an all-zero shared secret stops: the client is not recognised.
Both sides forget the keys when the connection ends.

## Sub ID 2: Secure Chat

| Field Name    | Field Type | Example | Notes                                      |
|---------------|------------|---------|--------------------------------------------|
| Packet ID     | UByte      | `66`    | Always `66`.                               |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet.            |
| Counter       | LE Uint    | `0`     | Starts at `0` in each direction, then +1.  |
| Message       | Byte[]     |         | The encrypted message, `n` bytes.          |
| Tag           | UByte[16]  |         | The authentication tag, ending the packet. |

The plaintext is a [Chat Message](../protocol075.md#chat-message) after its
Packet ID, ending with its NUL, and is handled as one. The server takes the
player id to be the sender's, and sends Secure Chat only for what is meant for
that player alone, such as a command's answer.

It is encrypted with ChaCha20-Poly1305
([RFC 8439](https://www.rfc-editor.org/rfc/rfc8439)), with the key of its
direction, the counter followed by eight zero bytes as the nonce, and the Packet ID, Sub
Packet ID and Counter as additional data. The receiver drops a message whose
counter is not above the last one it accepted, or whose tag does not verify. A
side whose counter would wrap stops sending.

Once the server has verified a Response, it sends a Secure Chat with counter
`0`, such as a system message saying the player is recognised. The client sends
Secure Chat only after receiving it.

## Server Use

What the server does with a key is its own business. It grants whatever rights
its operator gave that key. An admin key is one the operator added by hand,
copied from the player outside the game, so its login never crosses the
network. Which commands are privileged is the server's choice; from an admin
key it accepts them only through Secure Chat. Admins who log in another way are
unaffected.

Servers that trust each other, such as one organisation's, may share keys, admin keys included, so a
player or admin is recognised on all of them. Keys are public, so sharing them
lets nobody log in.

A key proves the same client install, not the same person. A server may offer
a way to unlink a key, such as `/logout`, decides which connection is recognised
when one key is on two, and may ignore a key and ask for a fresh login when
something looks wrong, such as a new network or location.

The rest of the connection is not encrypted: someone on the path can still play
as a recognised player, but cannot read or forge Secure Chat. The server is not
authenticated: someone posing as it can read what the player sends it, but
cannot log in elsewhere as them.

See [Extensions](extension.md) for how the extension is negotiated.
