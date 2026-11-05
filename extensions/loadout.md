# Loadout

The server names and describes the three classes, decides which items each
player carries and in which slots, and can retune a slot's fire interval,
spread and recoil while the game runs. It adds a pistol, a bolt-action rifle, a
double-barrelled shotgun, a belt-fed machine gun, a heal pistol, a grenade
launcher, a rocket launcher, a smoke grenade, a custom grenade, a bandage and
an empty hand to the base spade, block, rifle, SMG, shotgun and grenade.

| ------------: | ------------- |
| Extension ID: | `0x3B`        |
| Packet ID:    | `0x7B`        |
| Version:      | 1             |
| Type:         | `HAS_PACKETS` |

### Sub Packets:

| Sub ID | Name       | Direction         | Size |
|--------|------------|-------------------|------|
| 0      | Class      | Server -> Client  | 8+   |
| 1      | Loadout    | Server -> Client  | 5+   |
| 2      | Hold       | Client <-> Server | 4    |
| 3      | Ammo       | Server -> Client  | 5    |
| 4      | Modifier   | Server -> Client  | 21   |
| 5      | Projectile | Client <-> Server | 33   |
| 6      | Detonate   | Server -> Client  | 17   |
| 7      | Kill Item  | Server -> Client  | 5+   |

A server using this extension plays only with clients that have it. A client
that does not announce it in [ExtInfo](extension.md#extinfo-packet) is kept a
spectator or disconnected, as the server chooses, and is never sent a packet
of this extension.

**String** fields are NUL-terminated and encoded the way a
[Chat Message](../protocol075.md#chat-message) is: Code Page 437, or UTF-8
behind the `0xff` prefix of [UTF-8 Chat](utf-8-chat.md).

## Items

Every slot holds one item. Its numbers are fixed by this table; the server
changes them only through a slot's [Count and Reserve](#slot) and a
[Modifier](#sub-id-4-modifier). An item outside the table is held as an empty
hand.

| Value | Item                  | Kind       |
|-------|-----------------------|------------|
| 0     | Empty hand            | Tool       |
| 1     | Spade                 | Tool       |
| 2     | Block                 | Tool       |
| 3     | Bandage               | Tool       |
| 16    | Rifle                 | Gun        |
| 17    | SMG                   | Gun        |
| 18    | Shotgun               | Gun        |
| 19    | Pistol                | Gun        |
| 20    | Bolt-action rifle     | Gun        |
| 21    | Double-barrel shotgun | Gun        |
| 22    | Machine gun           | Gun        |
| 23    | Heal pistol           | Gun        |
| 32    | Grenade launcher      | Launcher   |
| 33    | Rocket launcher       | Launcher   |
| 48    | Frag grenade          | Throwable  |
| 49    | Smoke grenade         | Throwable  |
| 50    | Custom grenade        | Throwable  |
| other | reserved              |            |

### Defaults

Every value below is the item's default, damage included, and every server
starts from it, so that an item plays the same on every server and players
learn it once. A server changes counts through a [Slot](#slot) and firing
through a [Modifier](#sub-id-4-modifier), which the client shows. Damage is
dealt by the server alone, as today; one that departs from the defaults should
say so to its players.

Damage follows the base game, as LSD deals it:

* A Hit on a teammate does nothing, except the heal pistol's. Legs take the
  damage of arms.
* **Healing** raises HP by its amount, up to 100. The server sends the healed
  player a [Set HP](../protocol075.md#set-hp) with the new HP; a client draws
  no hurt for a Set HP that raises its HP.
* A Hit to the head is a `HEADSHOT` kill, a melee Hit a `MELEE` kill, any
  other Hit a `WEAPON` kill.
* An **explosion** deals `4096` divided by the squared distance in blocks to
  each living player within 16 blocks on every axis, with no block in between,
  to the enemy and to the thrower but not to the thrower's teammates. It
  removes the 3×3×3 blocks around it, as a base grenade's
  [Block Action](../protocol075.md#block-action) does, and is a `GRENADE`
  kill.

The default classes are the base game's, for both teams, each with its items'
default counts and no flags:

| Class | Name    | Description                         | Slots                                        |
|-------|---------|-------------------------------------|----------------------------------------------|
| 0     | Rifle   | `"Accurate at range. Slow to fire."` | Spade, Block 50, Rifle 10 + 50, Frag grenade 3 |
| 1     | SMG     | `"Fast to fire. Best up close."`    | Spade, Block 50, SMG 30 + 120, Frag grenade 3  |
| 2     | Shotgun | `"Eight pellets. Short range only."` | Spade, Block 50, Shotgun 6 + 48, Frag grenade 3 |

The slots are in the base game's tool order, so the usual keys `1` to `4` pick
the spade, block, gun and grenade.

### Tools

* **Empty hand**: nothing in hand. Primary fire punches: a melee
  [Hit](../protocol075.md#hit-packet) at the spade's reach, once every 0.5 s,
  that neither digs nor damages blocks. It deals 20 damage. Secondary fire does
  nothing.
* **Spade** and **Block**: as in the base game. A spade Hit deals 80 damage.
  Melee Hits are valid only with a spade or an empty hand held.
* **Bandage**: heals its holder. Holding primary fire for 2 s uses one bandage
  and heals 30; releasing or switching sooner uses none. It cannot be used at
  full HP. The server times the hold from
  [Weapon Input](../protocol075.md#weapon-input), as it times a gun's fire,
  then sends the Set HP and an [Ammo](#sub-id-3-ammo). Count 2 by default.
  Secondary fire does nothing.

### Guns

Guns cast rays as the base guns do and report hits with the
[Hit Packet](../protocol075.md#hit-packet). Each fires a round when primary
fire is pressed, then one per Fire interval while it is held, as the base guns
do: there is no semi-automatic mode, since servers count rounds from how long
fire is held. Pressing primary fire cancels a reload. Secondary fire aims.

Damage is per pellet. Spread, recoil and block damage are in the terms of the
0.75 game:
spread is added to each axis of the unnormalised aim, and recoil turns the aim
up and sideways as Voxlap does: the sideways kick is scaled by a triangle wave
running from `-255.5` to `255.5` and back every 1024 ms, both are doubled while
walking without aiming and while airborne, and halved while crouching. Block
damage is what each pellet takes off a block's
health, which the client counts. The first three rows are the base weapons'
values.

| Gun                   | Fire interval | Magazine | Reserve | Reload            | Pellets | Spread | Recoil up | Recoil side | Head | Torso | Arms, legs | Block damage |
|-----------------------|---------------|----------|---------|-------------------|---------|--------|-----------|-------------|------|-------|------------|--------------|
| Rifle                 | 0.5 s         | 10       | 50      | 2.5 s, whole      | 1       | 0.006  | 0.05      | 0.0001      | 100  | 49    | 33         | 50           |
| SMG                   | 0.1 s         | 30       | 120     | 2.5 s, whole      | 1       | 0.012  | 0.0125    | 0.00005     | 75   | 29    | 18         | 34           |
| Shotgun               | 1.0 s         | 6        | 48      | 0.5 s, per round  | 8       | 0.024  | 0.1       | 0.0002      | 37   | 27    | 16         | 34           |
| Pistol                | 0.25 s        | 8        | 40      | 1.5 s, whole      | 1       | 0.010  | 0.03      | 0.0001      | 75   | 34    | 25         | 25           |
| Bolt-action rifle     | 1.25 s        | 5        | 25      | 0.6 s, per round  | 1       | 0.002  | 0.1       | 0.0002      | 100  | 75    | 50         | 50           |
| Double-barrel shotgun | 0.2 s         | 2        | 24      | 2.0 s, whole      | 8       | 0.030  | 0.15      | 0.0003      | 37   | 27    | 16         | 34           |
| Machine gun           | 0.12 s        | 100      | 100     | 5.0 s, whole      | 1       | 0.016  | 0.02      | 0.0001      | 75   | 32    | 20         | 40           |
| Heal pistol           | 0.5 s         | 10       | 50      | 2.0 s, whole      | 1       | 0.006  | 0.03      | 0.0001      | 0    | 0     | 0          | 0            |

* **Pistol**: a handgun held in one hand.
* **Bolt-action rifle**: the bolt is worked after each round, and its
  animation fills the Fire interval. It is loaded one round at a time.
* **Double-barrel shotgun**: each round is one barrel. The gun breaks open to
  reload both barrels at once.
* **Machine gun**: a heavy belt-fed gun. Its magazine is the belt, and a reload
  opens the cover and lays in a new one.
* **Heal pistol**: fires healing darts. A Hit on a teammate heals them 20,
  whatever the body part; a Hit on an enemy, or on a block, does nothing. It
  cannot heal its holder.

#### Suppressor

Any gun whose slot is `SILENCED` carries a suppressor, drawn on its barrel by
every client. Its shots sound suppressed, are heard by other players only
within 32 blocks, and show no muzzle flash and no tracer. Nothing else
changes: damage, spread, recoil and fire interval stay the gun's. A server that
wants a trade-off uses a [Modifier](#sub-id-4-modifier).

The suppressor only changes what honest clients draw and play: every client
still receives the shooter's position and
[Weapon Input](../protocol075.md#weapon-input).

A reload starts when the client sends a
[Weapon Reload](../protocol075.md#weapon-reload), and does nothing when the
magazine is full or the reserve empty. A **whole** reload moves rounds from the
reserve into the magazine once its time has passed. A **per round** reload
moves one round each time its time passes, and goes on until the magazine is
full or the reserve empty, as the base shotgun does. Pressing primary fire or
switching slots cancels either.

### Launchers

A launcher fires a projectile, sent as a
[Projectile](#sub-id-5-projectile), instead of casting a ray. Its magazine
holds one round. Secondary fire aims.

| Launcher         | Magazine | Reserve | Reload        | Launch speed | Gravity | Bounces       | Fuse                     | Recoil up |
|------------------|----------|---------|---------------|--------------|---------|---------------|--------------------------|-----------|
| Grenade launcher | 1        | 6       | 2.0 s, whole  | 2            | Yes     | Unless IMPACT | 3 s, or 5 s with IMPACT  | 0.15      |
| Rocket launcher  | 1        | 3       | 3.0 s, whole  | 2            | No      | Never         | 5 s                      | 0.1       |

Launch speed is in [Grenade Packet](../protocol075.md#grenade-packet) units:
the projectile leaves along the aim at that speed, plus the player's velocity.
A thrown grenade leaves at speed `1`.

* **Grenade launcher**: break-barrel, single shot. Its round flies and bounces
  as a grenade does.
* **Rocket launcher**: single shot. Its rocket flies straight at a constant
  speed, and always detonates on impact.

Both detonate as a frag grenade does, with its
[explosion](#defaults), at the end of their fuse at the latest.

### Throwables

A throwable is thrown with primary fire as the base grenade is, and sent as a
[Projectile](#sub-id-5-projectile). Without `IMPACT` it is cooked while
primary fire is held and its fuse runs from the cook.

| Throwable      | Count | Fuse                     | Detonation                                                                    |
|----------------|-------|--------------------------|-------------------------------------------------------------------------------|
| Frag grenade   | 3     | 3 s, or 5 s with IMPACT  | An [explosion](#defaults), as the base grenade.                               |
| Smoke grenade  | 2     | 2 s, or 5 s with IMPACT  | A cloud in the slot's Colour, see below. No damage, no blocks broken.         |
| Custom grenade | 1     | 3 s, or 5 s with IMPACT  | A small burst in the slot's Colour. What else happens is the server's effect. |

* **Smoke grenade**: the cloud is centred where it detonates, grows to a
  radius of 5 blocks over 2 s, stays 20 s, then fades over 5 s. It hides what
  is behind it and does nothing else. The smoke is the slot's Colour, opaque
  whatever the colour, so a server can hand out red, green or white smoke. The
  default is grey, `128, 128, 128`. The canister carries a band of that colour.
* **Custom grenade**: a grenade in the slot's Colour, called by the slot's
  Name in the HUD and the kill feed. It does nothing on its own: on its
  [Detonate](#sub-id-6-detonate) the server applies whatever effect it chose,
  such as damage, a heal, a teleport or blocks, with the packets it already
  uses for them.

### Detonation

A projectile without `IMPACT` bounces as the base grenade does and detonates at
the end of its fuse, on each side's own clock, as today.

An `IMPACT` projectile, and every rocket, detonates at the first block or
player hitbox it touches, or at the end of its fuse. Only the server decides
where: it sends a [Detonate](#sub-id-6-detonate). A client that sees the
projectile touch something stops drawing it there, and draws its detonation
only when the Detonate arrives, at the Detonate's position.

Damage and broken blocks are always the server's, sent as
[Set HP](../protocol075.md#set-hp), [Kill Action](../protocol075.md#kill-action)
and [Block Action](../protocol075.md#block-action) as today.

## Slot

A slot is one item in a player's loadout or a class's, with what that item
carries.

| Field Name | Field Type | Example        | Notes                                    |
|------------|------------|----------------|------------------------------------------|
| Item       | UByte      | `50`           | See [Items](#items).                     |
| Flags      | UByte      | `0b1`          | Bit `0` `IMPACT`: a launcher round or throwable detonates on impact. Bit `1` `SILENCED`: a gun carries a [suppressor](#suppressor). Each is ignored on items it does not apply to. Bits `2`-`7` must be `0` and are ignored. |
| Count      | UByte      | `1`            | A gun's or launcher's magazine; how many blocks, bandages or throwables are carried. `255` is infinite. Ignored for the spade and empty hand. |
| Reserve    | UByte      | `0`            | A gun's or launcher's rounds beyond the magazine. `255` is infinite. `0` and ignored for other items. |
| Colour     | UByte[3]   |                | Blue, green, red, as in [Set Colour](../protocol075.md#set-colour). The smoke grenade's smoke and the custom grenade's colour; ignored for other items. |
| Name       | String     | `"Medic gas"`  | The custom grenade's name. Empty for other items, and ignored. |

The [Items](#items) tables give the default Count and Reserve; the server writes
the values it wants. A slot starts full: its magazine holds Count rounds and its
reserve Reserve.

An infinite magazine never empties and is never reloaded; a
[Weapon Reload](../protocol075.md#weapon-reload) for it is ignored. An infinite
reserve is never used up, so a reload is always allowed. An infinite count of
blocks, bandages or throwables never runs out. A client shows none of these
counts.

## Sub ID 0: Class

One of the three classes a player picks with
[Change Weapon](../protocol075.md#change-weapon): its name, its description and
the slots it spawns with, for the client to show before the player spawns.

| Field Name    | Field Type | Example                             | Notes                                       |
|---------------|------------|-------------------------------------|---------------------------------------------|
| Packet ID     | UByte      | `0x7B`                              | Always `0x7B`.                              |
| Sub Packet ID | UByte      | `0`                                 | Always `0` for this sub-packet.             |
| Team          | UByte      | `0`                                 | `0` or `1`. Each team has its own classes.  |
| Class         | UByte      | `2`                                 | `0`-`2`: the Weapon ID that picks it in Change Weapon. |
| Flags         | UByte      | `0b0`                               | Bit `0` `LOCKED`: shown, but cannot be picked. Bit `1` `HIDDEN`: not shown, and cannot be picked. Bits `2`-`7` must be `0` and are ignored. |
| Name          | String     | `"Breacher"`                        | Replaces Rifle, SMG or Shotgun in the class menu. |
| Description   | String     | `"Clears rooms. Short range only."` | Shown with the class. A line break is `\n`. |
| Slot Count    | UByte      | `4`                                 | Slots that follow.                          |
| Slots         | Slot[]     |                                     | See [Slot](#slot).                          |

The client shows a team's classes with their items and counts, so a player sees
what each class carries before choosing. A class the server has sent no Class
for is its [default](#defaults); a server that wants it gone sends it
`HIDDEN`.

The server sends every Class before [State Data](../protocol075.md#state-data),
and a new one whenever it changes a class, at any time. A new Class replaces the
previous one for that team and number, and the client's menu changes at once.
It applies from the next spawn: a player who is alive keeps their loadout, and
picks the new class when they next die or open the class menu.

A [Change Weapon](../protocol075.md#change-weapon) for a `LOCKED` or `HIDDEN`
class is ignored. Otherwise class changes are handled as in the base game,
which may kill a living player with `CLASS_CHANGE`.

## Sub ID 1: Loadout

One player's slots. The server sends it to every client, for every player, so that each can draw what others hold and how fast they fire.

| Field Name    | Field Type | Example | Notes                                      |
|---------------|------------|---------|--------------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                             |
| Sub Packet ID | UByte      | `1`     | Always `1` for this sub-packet.            |
| Player ID     | UByte      | `7`     | Whose loadout this is.                     |
| Class         | UByte      | `2`     | The class it comes from, `0`-`2`, or `255` for none. The class menu marks it. |
| Slot Count    | UByte      | `4`     | Slots that follow.                         |
| Slots         | Slot[]     |         | See [Slot](#slot).                         |

Slots are numbered from `0`, in the order the player cycles through them. A
client maps its keys to slots as it likes.

The server sends a player's Loadout before the
[Create Player](../protocol075.md#create-player) or
[Existing Player](../protocol075.md#existing-player) that introduces them, and
again whenever it changes their slots, at any time. At a spawn it is usually the
slots of the player's class, but the server may give any.

A new Loadout takes effect at once. A slot that holds the same Item as the slot
of the same number before keeps its counts, its reload and its
[Modifier](#sub-id-4-modifier), and takes its new Flags, Colour and Name: this
is how a server adds a suppressor or recolours smoke mid-game. Its counts are
cut down to the new Count and Reserve if they are over them. Every other slot
starts full, and its reload, cooking and modifier end.

The player keeps holding the same slot number if its Item is unchanged, and
otherwise holds slot `0` until a [Hold](#sub-id-2-hold) says otherwise; a
server that changes the held slot's item sends a Hold after the Loadout.

A respawn refills every slot. The player spawns holding their first gun or
launcher slot, or slot `0` when there is none.

## Sub ID 2: Hold

Which slot a player holds.

| Field Name    | Field Type | Example | Notes                                          |
|---------------|------------|---------|------------------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                                 |
| Sub Packet ID | UByte      | `2`     | Always `2` for this sub-packet.                |
| Player ID     | UByte      | `7`     | Ignored from a client; the server fills it in. |
| Slot          | UByte      | `2`     | An index into the player's slots.              |

A client sends Hold instead of [Set Tool](../protocol075.md#set-tool) when its
player switches. The server sends the Hold it settles on to every client, the
sender included. A Hold for a
slot beyond the loadout, or for a slot with nothing left to use, is refused:
the server sends the slot actually held back to the sender.

The server can switch a player at any time by sending a Hold for them to
everyone, the player included. The player's client switches and sends the same
Hold back, which the server does not relay. Until that echo arrives the server judges the player's Hits by the
slot they held before, since those were sent before the client switched.

Switching away cancels a reload, on both sides, and the server sends the player
an [Ammo](#sub-id-3-ammo) for the slot left, since their client may have
counted the reload as done. A cooked throwable is dropped as in the base game.

## Sub ID 3: Ammo

The server's count of one slot of the receiving player. It is sent to that
player only, and the client adopts it.

| Field Name    | Field Type | Example | Notes                                  |
|---------------|------------|---------|----------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                         |
| Sub Packet ID | UByte      | `3`     | Always `3` for this sub-packet.        |
| Slot          | UByte      | `2`     | An index into the player's slots.      |
| Count         | UByte      | `8`     | The magazine, or the blocks, bandages or throwables left. `255` is infinite. |
| Reserve       | UByte      | `40`    | The rounds beyond the magazine. `255` is infinite. |

The client counts what it uses on its own, and the server corrects it. The
server answers a [Weapon Reload](../protocol075.md#weapon-reload) with an Ammo
for the held slot when the reload ends, or after each round of a per-round
reload, instead of a Weapon Reload. It relays the Weapon Reload to the other
clients when the reload starts, as today, for the sound. It also sends an Ammo
when it changes a count itself, such as a pickup, and may send one whenever the
client is off.

An infinite count sent here lasts until the next Ammo, a Loadout that changes
the slot's Item, or a respawn: a
server can give infinite rounds for a while without changing the slot.

A [Restock](../protocol075.md#restock) refills every slot's reserve, blocks,
bandages and throwables, but not magazines, as the base Restock does, and needs no Ammo.

## Sub ID 4: Modifier

Retunes how one slot of a player fires, or all of them. It is sent to every
client, since others draw the player's shots at its fire
interval.

| Field Name    | Field Type | Example | Notes                                             |
|---------------|------------|---------|---------------------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                                    |
| Sub Packet ID | UByte      | `4`     | Always `4` for this sub-packet.                   |
| Player ID     | UByte      | `7`     | Whose slot it retunes.                            |
| Slot          | UByte      | `2`     | An index into the player's slots, or `255` for all of them. |
| Fire Interval | LE Float   | `0.5`   | Multiplies the slot's fire interval, or its throw delay. `0.5` fires twice as fast. |
| Spread        | LE Float   | `0`     | Multiplies the slot's spread. `0` is a perfect aim. |
| Recoil        | LE Float   | `1`     | Multiplies the slot's recoil.                     |
| Lasts         | UByte      | `2`     | `0` until replaced, `1` until the player dies, `2` for Duration. |
| Duration      | LE Float   | `10`    | Seconds, with Lasts `2`; ignored otherwise. |

Each factor must be finite and not negative, and Fire Interval greater than
`0`; a Modifier that breaks this is ignored. A factor that does not apply to the item, such as the spread of a
grenade, is ignored.

A slot has at most one modifier, and a new Modifier replaces it; `255` replaces
every slot's. A Modifier whose factors are all `1` clears it. A modifier also
ends when a [Loadout](#sub-id-1-loadout) changes its slot's Item, and with
Lasts `1` at the
player's [Kill Action](../protocol075.md#kill-action). Otherwise it survives
deaths and lasts until the server replaces or clears it. With Lasts `2` the
server clears it once Duration has passed on its own clock; the client uses
Duration only to show the time left, and keeps the modifier until it is cleared.

The server counts rounds and checks fire rate with the modifier applied.

## Sub ID 5: Projectile

A launcher round or a throwable, from the slot that fired it.

| Field Name    | Field Type | Example | Notes                                          |
|---------------|------------|---------|------------------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                                 |
| Sub Packet ID | UByte      | `5`     | Always `5` for this sub-packet.                |
| Player ID     | UByte      | `7`     | Ignored from a client; the server fills it in. |
| Slot          | UByte      | `4`     | The slot that fired it.                        |
| Projectile ID | UByte      | `12`    | Chosen by the thrower's client, see below.     |
| Fuse          | LE Float   | `2.4`   | Seconds left before it detonates.              |
| X position    | LE Float   | `0`     |                                                |
| Y position    | LE Float   | `0`     |                                                |
| Z position    | LE Float   | `0`     |                                                |
| X velocity    | LE Float   | `0`     |                                                |
| Y velocity    | LE Float   | `0`     |                                                |
| Z velocity    | LE Float   | `0`     |                                                |

A client sends a Projectile instead of a
[Grenade Packet](../protocol075.md#grenade-packet), and draws its own
projectile at once. It numbers its projectiles from `0`, counting up and
wrapping after `255`; a projectile is known by its player and Projectile ID.

The server checks it as it checks a Grenade Packet, against the slot held and
its count, then simulates it and relays it to the other clients. They take its
look, physics and detonation from the slot of the thrower's loadout.

A Projectile the server refuses is relayed to no one. The server sends the
thrower a [Detonate](#sub-id-6-detonate) marked `DUD` for it, and an
[Ammo](#sub-id-3-ammo) for the slot.

A base Grenade Packet sent by the server, such as an airstrike, is a frag
grenade.

## Sub ID 6: Detonate

The server detonates a projectile.

| Field Name    | Field Type | Example | Notes                                   |
|---------------|------------|---------|-----------------------------------------|
| Packet ID     | UByte      | `0x7B`  | Always `0x7B`.                          |
| Sub Packet ID | UByte      | `6`     | Always `6` for this sub-packet.         |
| Player ID     | UByte      | `7`     | Who fired it.                           |
| Projectile ID | UByte      | `12`    | As in its [Projectile](#sub-id-5-projectile). |
| Flags         | UByte      | `0b0`   | Bit `0` `DUD`: remove it without a detonation. Bits `1`-`7` must be `0` and are ignored. |
| X position    | LE Float   | `0`     | Where it detonates.                     |
| Y position    | LE Float   | `0`     |                                         |
| Z position    | LE Float   | `0`     |                                         |

The server sends it to every client, the thrower included,
for every `IMPACT` projectile and rocket, and for any projectile it detonates
before its fuse ends. The client detonates the projectile there, whether or not
it is still drawing it, or with `DUD` removes it without a detonation. A Detonate for a projectile the client has already
detonated is ignored.

## Sub ID 7: Kill Item

The item that made the next kill.

| Field Name    | Field Type | Example       | Notes                                    |
|---------------|------------|---------------|------------------------------------------|
| Packet ID     | UByte      | `0x7B`        | Always `0x7B`.                           |
| Sub Packet ID | UByte      | `7`           | Always `7` for this sub-packet.          |
| Player ID     | UByte      | `3`           | The player who died.                     |
| Item          | UByte      | `50`          | See [Items](#items).                     |
| Name          | String     | `"Medic gas"` | The custom grenade's name, else empty.   |

The server sends it to every client just before the
[Kill Action](../protocol075.md#kill-action) of that player, which then shows
the item's icon, or the Name, in the kill feed. It applies to that Kill Action
only. A Kill Action without one shows the icon of its kill type, as today.

## Base packets

* **Change Weapon**'s Weapon ID picks a [Class](#sub-id-0-class).
* **Create Player** and **Existing Player** carry the player's class number,
  or `0` with no class, as Weapon. A client takes what the player carries from
  their Loadout, not from Weapon, and ignores the held item of Existing
  Player, which a [Hold](#sub-id-2-hold) gives.
* **Set Tool** is not sent by either side; one received is ignored.
* **Weapon Input** fires the held slot.
* **Hit Packet** is checked against the held slot's item. A Hit while holding a
  launcher, a throwable or a block is ignored, and so is any Hit but a melee
  one while holding an empty hand. Any Hit while holding a bandage is ignored.
* **Weapon Reload** from the client reloads the held slot, and is answered with
  an [Ammo](#sub-id-3-ammo).
* **Restock** refills every slot's reserve, blocks, bandages and throwables, not
  magazines.
* **Grenade Packet** from a client is ignored; it sends a
  [Projectile](#sub-id-5-projectile).

## Validation

A client parses a whole Class or Loadout before applying it, and refuses it if
a slot or string runs past the end of the packet, or bytes remain after the
last slot. A refused packet leaves the previous one in place and is logged, not
a reason to disconnect. A packet naming a slot the player does not have is
ignored.

## Connection state

Classes, loadouts, held slots and modifiers belong to the world.
[Map Start](../protocol075.md#map-start-075) clears them all; the server sends
the Classes again before the next State Data, and each player's Loadout again
before introducing them. [Player Left](../protocol075.md#player-left) clears
that player's loadout, held slot and modifiers.

See [Extensions](extension.md) for how the extension is negotiated.
