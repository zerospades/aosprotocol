# aosprotocol
Documentation and development of the protocol used primarily by Voxlap (classic) v0.75/v0.76, OpenSpades, BetterSpades, pyspades, PySnip and piqueserver.

This started out as a dump of the aoswiki.rakiru.com Protocol page, but is meant to be the hub for
developing the AoS protocol in the future.

Browse the documentation at https://www.piqueserver.org/aosprotocol, or just read through the git repository instead.

## Other references

* [AoS 0.75.015 Canon Client Doc](https://notaburner.mooo.com/assets/clidoc_draft.html)
  by not a burner (CC0): how a client should behave, packet by packet, with the
  conventions, character sets and world rules a client needs.

## Implementations
Active means a commit in the last twelve months, as of September 2026.

### Clients

| Name                                                       | Based on     | Language | Status   |
|------------------------------------------------------------|--------------|----------|----------|
| Voxlap (classic) v0.75/v0.76                               |              |          | Inactive |
| [OpenSpades](https://github.com/yvt/openspades)            |              | C++      | Inactive |
| [ZeroSpades](https://github.com/zerospades/zerospades)     | OpenSpades   | C++      | Active   |
| [IV of Spades](https://github.com/VierEck/openspades)      | OpenSpades   | C++      | Active   |
| [BetterSpades](https://github.com/xtreme8000/BetterSpades) |              | C        | Inactive |
| [TigerSpades](https://github.com/rzrn/tigerspades)         | BetterSpades | C        | Active   |
| [ButterSpades](https://github.com/utf-4096/butterspades)   | BetterSpades | C        | Archived |
| [KyroSpades](https://github.com/Kyrope01/KyroSpades)       | ButterSpades | C        | Active   |
| [ace](https://github.com/10se1ucgo/ace)                    |              | C++      | Inactive |

### Servers

| Name                                                      | Based on | Language | Status   |
|-----------------------------------------------------------|----------|----------|----------|
| pyspades                                                  |          | Python   | Inactive |
| [PySnip](https://github.com/NateShoffner/PySnip)          | pyspades | Python   | Inactive |
| [piqueserver](https://github.com/piqueserver/piqueserver) | PySnip   | Python   | Active   |
| [SpadesX](https://github.com/SpadesX/SpadesX)             |          | C        | Active   |
| [LSD](https://66.135.15.57/lsd/)                          |          | C        | Active   |

### Libraries

| Name                                                | Language   | Status   |
|-----------------------------------------------------|------------|----------|
| [libspades](https://notaburner.mooo.com/libspades/) | C          | Inactive |
| [AoS.js](https://github.com/DryByte/AoS.js)         | JavaScript | Inactive |

## Workflow
You want to add a packet to AoS? Great!

First check whether a chat [command](commands.md) would do: it needs no packet, and every client can already send one.

### Packet Proposal
File an issue and name it `PP: PacketName`. Describe what packet you wish to add,
and how it would improve the game.

### Packet Add Request
If your packet has recieved positively it is time to write it up.
Describe the packet in the style of the other packets and file a Pull Request and name it `PAR: PacketName`.
