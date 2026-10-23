# Cosmetic Weapon and Tool

The server sets what a player's tools look like: a bolt-action instead of the
rifle, a stick grenade instead of the vanilla one. A look never changes how a
tool behaves: the gun fires, the grenade flies and the spade digs as their base
tools do.

| ------------: | ------------- |
| Extension ID: | `0x3A`        |
| Packet ID:    | `0x7A`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name | Direction        | Size |
|--------|------|------------------|------|
| 0      | Look | Server -> Client | 5+   |

## Sub ID 0: Look

One player's looks. The server sends it to every client, for every player, once
after their [Existing Player](../protocol075.md#existing-player) or first
[Create Player](../protocol075.md#create-player), and then only when it changes
them. A server receiving one drops it.

| Field Name    | Field Type | Example | Notes                           |
|---------------|------------|---------|---------------------------------|
| Packet ID     | UByte      | `0x7A`  | Always `0x7A`.                  |
| Sub Packet ID | UByte      | `0`     | Always `0` for this sub-packet. |
| Player ID     | UByte      | `7`     | Whose looks these are.          |
| Entry Count   | LE UShort  | `2`     | Entries that follow.            |
| Entries       | Entry[]    |         | See below.                      |

| Entry Field | Field Type | Example | Notes                                        |
|-------------|------------|---------|----------------------------------------------|
| Slot        | LE UShort  | `3`     | Which slot, see [Slots](#slots).             |
| Model       | UByte      | `2`     | See [Models](#models).                       |
| Flags       | UByte      | `0b1`   | Bit `0` `TINTED`: multiply the model's colours by Colour. Bits `1`-`7` must be `0` and are ignored. |
| Colour      | UByte[3]   |         | Blue, green, red, as in [Set Colour](../protocol075.md#set-colour). Used only with `TINTED`; a block slot keeps its Set Colour, which a tint does not change. |

### Slots

A look is set on a slot, and the slot's mechanics come from elsewhere. Without
a [Loadout](loadout.md) a player has four slots, one per base tool: slot `n` is
the tool `n` of [Set Tool](../protocol075.md#set-tool), so the gun is slot `2`
and the grenade slot `3`. With a Loadout, its slots, in its order: a rifle in
slot `3` can look like a rifle and another rifle in slot `4` like a pistol.

A slot with no entry keeps its vanilla look, and a Look replaces the player's
previous one whole. The server sends a player's Loadout before their Look; a
new Loadout resets every look to vanilla until the next Look, and an entry for a
slot the player does not have is ignored. A Look can change a skin at any time
without resending the Loadout.

A client without the Loadout extension knows only the four base slots, so the
server sends it a Look in base-tool numbering: the look of the player's held
slot on that slot's base tool.

## Models

One list for every slot: any slot can be drawn as any model, whatever its tool.
`0` is the vanilla look of the slot's tool; `1` to `7` are the tools every
OpenSpades-family client already has; the rest are new, and a client without
one draws it as its fallback.

| Value     | Model          | Looks and animates like                           | Fallback |
|-----------|----------------|---------------------------------------------------|----------|
| `0`       | Vanilla        | The slot's own tool.                              |          |
| `1`       | Spade          | The vanilla spade.                                |          |
| `2`       | Pickaxe        | A pick, swung like the spade.                     | Spade    |
| `3`       | Block          | The vanilla block.                                |          |
| `4`       | Grenade        | The vanilla grenade.                              |          |
| `5`       | Rifle          | The vanilla rifle.                                |          |
| `6`       | SMG            | The vanilla SMG.                                  |          |
| `7`       | Shotgun        | The vanilla shotgun.                              |          |
| `8`       | Bolt-action    | A long rifle whose bolt is worked between rounds. | Rifle    |
| `9`       | Pistol         | A handgun, held in one hand.                      | Rifle    |
| `10`      | Machine gun    | A heavy belt-fed gun.                             | SMG      |
| `11`      | DMR            | A scoped semi-automatic marksman rifle.           | Rifle    |
| `12`      | Launcher       | A tube that fires rockets, see below.             | Shotgun  |
| `13`      | Stick grenade  | A grenade on a wooden handle.                     | Grenade  |
| `14`      | Smoke canister | A cylinder, as a smoke grenade.                   | Grenade  |
| `15`      | Rocket         | A finned rocket.                                  | Grenade  |
| `16`-`255`| reserved       |                                                   | Vanilla  |

An animation between rounds, such as working a bolt, lasts the time between
rounds of the gun underneath.

A thrown grenade keeps the look of the slot that threw it, colour included, so a server
can make any grenade it wants: a red smoke canister, a white flash, a green gas.
Its explosion is the base grenade's whatever the look.

A [Grenade Packet](../protocol075.md#grenade-packet) the server sends for a
player holding a slot whose model is `Launcher` is drawn as a `Rocket`,
tinted as that slot. This lets a server fire rockets from a launcher; they still
fly as grenades do. The rule covers every Grenade Packet sent for that player
while the launcher is held, an airstrike included, so a server that does not
want rockets there sends those for no player. A look changes nothing, so the
gun underneath still fires as its base weapon unless the server stops it.

A model is the tool's look, animation, sounds and kill icon. The kill icon is
that of the slot the killer holds when [Kill Action](../protocol075.md#kill-action)
arrives, or of their grenade slot for a grenade kill. Clients without the
extension draw the vanilla tools.

## Connection state

A player's looks last until another Look replaces them or
[Player Left](../protocol075.md#player-left). A respawn and
[Map Start](../protocol075.md#map-start-075) keep them.

See [Extensions](extension.md) for how the extension is negotiated.
