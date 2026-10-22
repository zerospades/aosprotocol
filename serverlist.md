# Server List

Clients find public servers by downloading a JSON list over HTTP. Servers get
onto it through the [Master Server Protocol](protocolmaster.md); this page is
the other side, what a client reads.

## Fetching

```
GET /serverlist.json
```

over HTTP/1.0 or later, with a `Host` header. Known hosts:

| Host                         | Notes                                                       |
|------------------------------|-------------------------------------------------------------|
| `services.buildandshoot.com` | The original list. Plain HTTP works.                        |
| `aos.coffee`                 | Merges several lists, including the one above. Redirects to HTTPS. |

## Format

A UTF-8 JSON array with one object per server:

```json
{
  "name": "Deuces Wild",
  "identifier": "aos://16777343:32887",
  "map": "hallway",
  "game_mode": "ctf",
  "country": "US",
  "latency": 98,
  "players_current": 16,
  "players_max": 32,
  "last_updated": 1792698008,
  "game_version": "0.75"
}
```

| Key               | Type   | Meaning                                                        |
|-------------------|--------|----------------------------------------------------------------|
| `name`            | string | The server's name.                                             |
| `identifier`      | string | Its [`aos://` address](protocol075.md#server-addresses).       |
| `map`             | string | The current map.                                               |
| `game_mode`       | string | The mode name the server reports, free text such as `ctf` or `WAR`. It is not the [Gamemode ID](protocol075.md#gamemode-id) a client receives. |
| `country`         | string | ISO 3166-1 alpha-2 code, from the server's IP address.         |
| `latency`         | number | Round trip between the list and the server, in milliseconds.   |
| `players_current` | number | Players connected.                                             |
| `players_max`     | number | Player slots.                                                  |
| `last_updated`    | number | When the entry last changed, in seconds since the Unix epoch.  |
| `game_version`    | string | `"0.75"`, `"0.76"`, or another string.                         |

A client should ignore keys it does not know, since more may be added. The same
server can appear more than once; clients do not merge duplicates.

Most clients show their own ping instead of `latency`, by
[pinging](protocolping.md) each server themselves.
