This page documents the networking Protocol of Ace of Spades 0.75, as well as
the extensions made by the community.

# Overview
[Ace of Spades](http://buildandshoot.com/) uses the [ENet networking
library](http://sauerbraten.org/enet/) for all server-client
communication. The initial source for the protocol information was the original
[pyspades](http://code.google.com/p/pyspades/) source code, for which the
source for 1.0 was not released. Nonetheless, the 1.0 alpha client has been
reverse engineered to document the protocol.

## Versions

 * [0.75 (and 0.76) documentation](protocol075.md)
 * [1.0 alpha documentation](protocol100a1.md)
 * [Client conventions](client.md)

## Extensions

The 0.75 protocol supports extensions, which allow adding new packets as well as
querying the support for client and server functionality.

The ids, types and negotiation mechanism are described in
[Extensions](extensions/extension.md). The small rules every extension follows, such as
NUL-terminated strings in CP437 or UTF-8, are in the [Standard](standard.md). The registry of extension ids, each
with a link to its specification, is listed in the
[0.75 documentation](protocol075.md#extensions).

### Implementers
 * [OpenSpades](https://github.com/yvt/openspades)
 * [piqueserver](https://github.com/piqueserver/piqueserver)
 * [BetterSpades](https://github.com/xtreme8000/BetterSpades)
 * [TigerSpades](https://github.com/rzrn/tigerspades)

Links to the respective projects pages that detail the extensions evailable in
each version should be linked here.

## Other Protocols

 * [Master Server Protocol](protocolmaster.md)
 * [Server List](serverlist.md)
 * [Ping Protocol](protocolping.md)

# Other Resources
* [KVX File Format Specification](https://web.archive.org/web/20100102023608/http://mystaddict.tlayeh.com/Computer%20Camp/Slab6/slab6.txt) - An archive of the mirror of the readme for Slab6 which contains the .kvx file format, the format that the AoS model format is based on
* [VXL File Format Specification](mapformat.md) - A description of the .vxl file format, the format used for AoS maps<br />([original](http://silverspaceship.com/aosmap/aos_file_format.html), [mirror](aos_file_format.html))
