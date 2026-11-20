# Client Conventions

Things players expect every client to do alike. None of them are on the wire,
but each ends in a packet the server sees.

## Block Palette

While the block tool is held, clients show an 8 by 8 palette of block colours.
Choosing a square sends a [Set Colour](protocol075.md#set-colour). Each row is
one hue, going from dark to light:

| Row | Hue     | Colours, left to right                                                    |
|-----|---------|---------------------------------------------------------------------------|
| 1   | Grey    | `0f0f0f 2f2f2f 4f4f4f 6f6f6f 8f8f8f afafaf cfcfcf efefef`                 |
| 2   | Red     | `1f0000 5f0000 9f0000 df0000 ff1f1f ff5f5f ff9f9f ffdfdf`                 |
| 3   | Orange  | `1f0f00 5f2f00 9f4f00 df6f00 ff8f1f ffaf5f ffcf9f ffefdf`                 |
| 4   | Yellow  | `1f1f00 5f5f00 9f9f00 dfdf00 ffff1f ffff5f ffff9f ffffdf`                 |
| 5   | Green   | `001f00 005f00 009f00 00df00 1fff1f 5fff5f 9fff9f dfffdf`                 |
| 6   | Cyan    | `001f1f 005f5f 009f9f 00dfdf 1fffff 5fffff 9fffff dfffff`                 |
| 7   | Blue    | `00001f 00005f 00009f 0000df 1f1fff 5f5fff 9f9fff dfdfff`                 |
| 8   | Magenta | `1f001f 5f005f 9f009f df00df ff1fff ff5fff ff9fff ffdfff`                 |

Colours are `rrggbb`. The fourth grey, `6f6f6f` or `111, 111, 111`, is the colour
a player spawns with. The arrow keys move the selection; the palette marks a
square only when the held colour is exactly its colour.

## Picking a Colour

A player can copy the colour of the block under the crosshair, within the
[fog](protocol075.md#about-coordinates), and hold it as their block colour. The
client sends a [Set Colour](protocol075.md#set-colour) for it.

## Sectors

The map and minimap label places by [sector](protocol075.md#about-coordinates),
such as `C4`, so that players can name a place in chat.
