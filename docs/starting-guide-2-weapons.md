# WormBench starting guide, part 2: Weapons

[Part 1](starting-guide-1-your-first-mod.md) built the Moon Grenade. Part 2
goes through a weapon component in full: its data and how a profile changes
it, the `ctx` table its `install` receives, and what `wa.weapons.replace`
can change. It adds a second weapon to the mod and gives the Moon Grenade an
icon of its own.

The new weapon is the **Scatter Bazooka**: one shot fires three smaller
rockets in a fan. The finished mod is
[examples/starting_guide](https://github.com/EdLud/WormBench-mods/tree/main/examples/starting_guide)
in the mods repository.

Tutorial Overview:

- A weapon's **data** (ammo, delay, crates, what the turn does after a use)
  is its `defaults`, with the profile's settings laid over them field by
  field. `install` reads the result as `ctx.data`.
- `ctx` tells `install` which weapon it builds (`ctx.id`), where the profile
  put it, and what else the profile has.
- `wa.weapons.replace` starts from a stock weapon (`copy_from`) or a Project
  X weapon file (`wep`), then changes its record, its name, its panel icon,
  its art and what it does.
- To change the numbers, write the record. To add to the burst, use
  `on_explode`. To change the shot itself, use `on_fire`: it runs in place of
  WA's projectile.
- `wa.weapons.launch` fires a real WA missile from Lua. With
  `weapon = ctx.id`, the missile is one of your weapon's own.

## The Scatter Bazooka

The mod gains a weapon file, an icon and a profile:

```text
starting_guide/
  mod.toml
  weapons/
    moon_grenade.lua
    scatter_bazooka.lua        new
  sprites/
    moon_grenade.png           new: the Moon Grenade's icon
  profiles/
    Moon Grenade.toml
    Scatter Bazooka.toml       new
```

`mod.toml` gains a component:

```toml
[[components]]
id = "guide.scatter_bazooka"
type = "weapon"
name = "Scatter Bazooka"
description = "A bazooka that fires three smaller rockets in a fan."
entry = "weapons/scatter_bazooka.lua"
icon = "bazooka"
default_position = { row = "F1", column = 6 }
```

### The weapon file

```lua
-- Scatter Bazooka: a bazooka that fires three smaller rockets in a fan.

local SPREAD = math.rad(10) -- between two rockets

return {
  defaults = {
    ammo = { round_start = 2 },
    delay = 2,
  },

  install = function(ctx)
    -- ctx.data: the defaults above, with the profile settings over them.
    local p = ctx.position
    wa.log(("on %s, column %d: ammo %d, delay %d"):format(
      p.row, p.column, ctx.data.ammo.round_start, ctx.data.delay))

    wa.weapons.replace{
      copy_from = "stock.bazooka",
      missile = { explosion = { power = 30 } },

      -- Runs in place of the rocket the Bazooka fires.
      on_fire = function(fire)
        for i = -1, 1 do
          local c, s = wa.math.cos(i * SPREAD), wa.math.sin(i * SPREAD)
          wa.weapons.launch{
            weapon = ctx.id, -- this weapon: power 30, and its callbacks
            owner = fire.worm,
            x = fire.x,
            y = fire.y,
            vx = fire.vx * c - fire.vy * s,
            vy = fire.vx * s + fire.vy * c,
          }
        end
      end,
    }
  end,
}
```

- `defaults`: two Scatter Bazookas at the start of a round, and two turns'
  delay before the first can be used.
- `install(ctx)` logs where the profile put the weapon and what its data came
  to. Under the profile of this part, `wkLua.log` reads
  `[guide.scatter_bazooka] on F1, column 6: ammo 2, delay 0`: the ammo comes
  from the defaults, the delay from the profile.
- `copy_from = "stock.bazooka"` with `missile.explosion.power = 30`: the stock
  Bazooka with a smaller blast.
- `on_fire` runs when the worm fires, in place of the Bazooka's rocket.
  `fire.x` and `fire.y` are where WA would have put the rocket; `fire.vx` and
  `fire.vy` are its speed, in pixels per frame, from the aim and the charge.
- `wa.weapons.launch` fires three rockets from there: the middle one at that
  speed, the other two turned 10 degrees each way. `weapon = ctx.id` makes
  each one a Scatter Bazooka rocket, flying with this weapon's record: its
  blast has power 30, and the weapon's `on_explode`, if it had one, would run
  for it. `owner = fire.worm` makes the rockets the firing worm's.
- The `c` and `s` lines turn the speed by an angle. The result reaches the
  game, so they use `wa.math.cos` and `wa.math.sin`, not `math.cos`
  ([part 1](starting-guide-1-your-first-mod.md#3-the-simulation-changes-the-game-drawing-only-looks),
  rule 3). `math.rad` only multiplies, which is exact on every machine.

## A weapon's data

### Defaults, then the profile

A weapon's data starts from its `defaults`. A profile sets values in
`[weapon_settings."<id>"]`; the loader's **Manage Weapon Panel** writes them
there. Each setting in the profile replaces that field, and every field it
leaves out keeps the default. `install` reads the result as `ctx.data`.

This part's profile changes one Scatter Bazooka setting:

```toml
[weapon_settings."guide.scatter_bazooka"]
delay = 0
```

So the match has `ammo.round_start = 2`, from the defaults, and `delay = 0`,
from the profile.

- A field that neither sets is `nil` in `ctx.data`, and WA's value stands. A
  weapon of yours without `ammo.round_start` has infinite ammo, and one
  without a crate weight has 20, a WSC crate probability of 1.
- `ctx.data` is for reading. WormBench writes the settings into the game
  before any `install` runs, so changing `ctx.data` changes nothing.
- Stock weapons take settings the same way: part 1's profile gives
  `stock.ninja_rope` infinite ammo.

### Every field

`defaults` and a profile name a few fields differently: `defaults` keep the
ammo fields in one table, and a profile sets crate odds as a share. The
loader row is what a player sees in **Manage Weapon Panel**.

| In `defaults` | In a profile | Loader row | What it is |
|---|---|---|---|
| `ammo.round_start` | `ammo` | Ammo at round start | Ammo at the start of a round, up to 127. `-1` is infinite. |
| `ammo.turn_start = { amount, mode }` | `each_turn = { amount, mode }` | Each turn | Ammo the team gets at the start of each of its turns: `mode = "add"` adds `amount`, `"replace"` sets the ammo to it. |
| `ammo.crate_collect` | `crate_collect` | Ammo per crate | Ammo from one crate, up to 99. `-1` is infinite. |
| `ammo.stockpiling` | `stockpiling` | Between rounds | `"off"`: each round starts with `round_start` again. `"on"`: it adds to what was left. `"anti"`: what was left is all there is. |
| `delay` | `delay` | Delay (turns) | Turns before the weapon can be used. |
| `crate.chance_pct` | `crate_pct` | Crate chance | How often it comes in a crate. `defaults` set a weight against the other weapons of its kind: 20 is a WSC crate probability of 1, 100 is 5, the most the WA scheme editor offers, and 0 is never. A profile sets a share of those crates, in %. |
| `crate.type` | `crate_type` | Comes in | `"weapon"`, `"utility"` or `"super"` crates. |
| `after_use` | `after_use` | After use | What the turn does after a use: `"end"` ends it at once, `"retreat"` ends it after the retreat time, `"continue"` goes on. |
| `shots` | `shots` | Shots per use | 1 to 9 shots for one use, as the two of the stock Shotgun. |
| `remembered` | `remembered` | Remember Weapon | The worm gets the weapon back after a use, and holds it at the start of its next turn. |
| `auto_use` | `auto_use` | Auto-Use | A utility from a crate switches on at once; `false` keeps it to use by hand. |
| `donor_card` | `donor_card` | Goes into donor card | A team that dies leaves its ammo of this weapon in its donor card. |
| `ends_invisibility` | `ends_invisibility` | Ends Invisibility | Using it makes an invisible team visible again. |
| `mid_air` | `mid_air` | | It fires from the Ninja Rope, Bungee, Parachute or Jet Pack. |
| `airborne` | `airborne` | | It fires while the worm jumps or falls. |
| `placement_range` | `placement_range` | | While the weapon is selected, the cursor stays within this many pixels of the worm. |

A few fields belong to the weapon alone, and no profile changes them:

| In `defaults` | What it is |
|---|---|
| `placement` | `"girder"` or `"girder_pack"`: placed as the stock Girder or Girder Starter Pack. |
| `cursor_sprite` | One of WA's sprites, drawn at the cursor while the worm aims, as the Teleport draws a worm there. |
| `cursor = { flip, dy }` | The team cursor while the worm aims: upside down with `flip = true`, and `dy` pixels lower. |
| `aims_straight_up` | `false` keeps the aim a little short of straight up, as the Mortar does. |
| `collects_crates` | `false`: its missiles leave crates where they are. |

The Ninja Rope has `rope_length` and `rope_angle`, and the instant utilities
four fields of their own. The [verb list](verbs.md#weapon-data) has every
field.

## What ctx holds

`install` receives `ctx`:

| Field | What it holds |
|---|---|
| `ctx.id` | The weapon's id: `"guide.scatter_bazooka"`. |
| `ctx.bundle` | The mod's id: `"example.starting_guide"`. |
| `ctx.type` | `"weapon"`. A feature's `install` gets `"feature"`. |
| `ctx.position` | Where the profile put the weapon: `{ row = "F1", column = 6 }`. `row` is `"F1"` to `"F12"`, or `"utility"`. |
| `ctx.data` | The weapon's data: its defaults with the profile's settings over them. |
| `ctx.weapons` | Every weapon the profile places, stock ones too, by id: `ctx.weapons["stock.bazooka"].position` and `.data`. |
| `ctx.placed` | The same weapons as a list, each with its `id`, in the same order on every machine. |
| `ctx.selected(id)` | `true` when the profile places that weapon or selects that feature. |
| `ctx.option(key)`, `ctx.options` | The weapon's options, which part 3 brings in. |

- Where the code names its own weapon, write `ctx.id`. The file then keeps
  working in a copy of the mod under new ids (part 1,
  [To change someone else's mod](starting-guide-1-your-first-mod.md#to-change-someone-elses-mod)).
- `ctx.weapons` is a table by id, so `pairs()` walks it in a different order
  on each machine. Where the order reaches the game, walk `ctx.placed` with
  `ipairs`.

## What wa.weapons.replace changes

`install` calls `wa.weapons.replace` to build its weapon. It names no slot:
it builds the weapon the profile placed, and only that one. Its fields come
in five groups:

| Group | Fields |
|---|---|
| The base | `copy_from = "stock.<name>"`, or `wep = "<file>.wep"` |
| Name and icon | `name`, `panel_icon` |
| The record | `fires`, `handling`, `missile`, `clusters`, `mine`, `gun`, `strike`, ... |
| What it does | `on_fire`, `on_click`, `on_explode`, `on_place`, `on_proximity`, ... |
| How it looks | `sprite`, `hold`, `worm_sprites`, ... |

The [verb list](verbs.md#claim-fields) names every field.

### The base

`copy_from = "stock.<name>"` starts from a stock weapon: how WA fires it, how
its projectile flies and bursts, its sounds, its icon. Pick the stock weapon
closest to what you want: a grenade to throw, a bazooka to fire, dynamite to
set down. [Weapon records](weapon-records.md#start-with-an-existing-weapon)
lists the stock weapons by family.

`wep = "weps/<name>.wep"` starts from a Project X weapon file in your mod
instead, and the record it holds. The tables you write in `replace` still go
over it. A weapon has one base: `copy_from` or `wep`, never both.

### Name

The game shows the weapon under its `name` in `mod.toml`, the name the loader
shows too. A `.wep` file names its weapon, and the name in `mod.toml` takes
its place. `replace` takes a `name` as well, which wins in the game: leave it
out, and the loader and the game agree.

### An icon of its own

Part 1 left the Moon Grenade with the icon of the stock Grenade. One line in
its `replace` adds its own icon:

```lua
    wa.weapons.replace{
      copy_from = "stock.grenade",
      panel_icon = "sprites/moon_grenade.png",
```

One line in its `mod.toml` block shows the same icon in the loader:

```toml
icon = "sprites/moon_grenade.png"
```

- `panel_icon` is a PNG in your mod, 32×32, the size of the stock icons. The
  panel shows the middle 24×24 of it, so keep what matters 4 pixels from
  the edge. A PNG of up to 24×24 works too.
- The panel shows it in full colour, a little dimmer except under the
  pointer, as it shows the stock icons. It's checkered while the weapon
  can't be used yet. On a renderer other than Direct3D 9 (shader), WormBench
  fits its colours to those of the stock icons.
- `panel_icon = "banana"` borrows the icon of a stock weapon instead, by its
  name. Without `panel_icon`, a weapon shows the icon of the weapon it copies:
  the Scatter Bazooka shows the Bazooka's.
- `icon` in `mod.toml` is only the loader icon.

### The record

The record is how WA describes a weapon: how it fires, how its projectile
flies and bursts. `replace` writes it as named tables, over the base:

```lua
missile = {
  gravity_pct = 35,
  explosion = { power = 20 },
},
```

Everything you don't name keeps the value of the base.
[Weapon records](weapon-records.md) lists every table and field by family:
`missile` and `clusters` for what grenades and bazookas fire, `gun` for the
Shotgun and the Uzi, `mine`, `strike` and more. WormBench refuses a field the
weapon's family doesn't have, and the error names it.

`wa.weapons.data(id)` reads a record back as the same tables, in a callback
once the match runs: `wa.weapons.data("stock.bazooka").missile.explosion.power`
is the blast of the stock Bazooka. The table is a copy; changing it changes
no weapon.

## Record, on_explode or on_fire?

| You want | Write |
|---|---|
| The stock weapon with other numbers: gravity, blast, fuse, bounce, wind | The record: `missile = { ... }` |
| Something more where it bursts | `on_explode`, as the craters of the Moon Grenade |
| A different shot: more than one missile, or something that isn't a missile at all | `on_fire`, as the Scatter Bazooka |

Start with the record, and add a callback for what the record can't say. WA
flies, draws and bursts a missile from the record, or from
`wa.weapons.launch`, as it does any other.

What `on_fire` replaces depends on how the base fires:

- A weapon that fires a projectile (a bazooka, a grenade, dynamite set down):
  the projectile.
- A gun (the Shotgun, Uzi, Minigun or Handgun): every bullet. `fire.vx` and
  `fire.vy` are then the direction of the bullet, of length 1: multiply them
  by the speed you want.
- A utility, the Jet Pack or Skip Go: what WA does for it.

The rest of the shot stays as WA has it: the worm fires, the ammo goes down,
and the turn ends as `after_use` sets.

The `fire` table has `x`, `y`, `vx`, `vy`, `worm` (the worm firing),
`weapon` (its id) and `power` (how far the shot was charged, 0 to 1); the
verb list has the rest.

`wa.weapons.launch` works in simulation callbacks only, `on_fire` among them. It
takes `weapon`, `owner`, `x`, `y`, `vx` and `vy`, and returns the missile's
id. Its `missile = { ... }` and `clusters = { ... }` change the record for
that one missile, as `missile = { explosion = { power = 60 } }` for a heavier
middle rocket. The weapon must fire a missile: one launched, thrown or set
down. `weapon = "stock.<name>"` fires the stock weapon.

> **Coming from Project X.** PyroMan's Nuke Rocket takes over its shot in
> PX's terms:
>
> ```cpp
> override void CWorm::FireFinal(CWeapon* Weap,CShootDesc* Desc)
> {
>     if (Weap->CheckName("Nuke Rocket"))
>     {
> // ...
>         SDesc.SpX = Desc->SpX;
>         SDesc.SpY = Desc->SpY;
> // ...
>         ldata->launch.explosion.damage = 0;
> // ...
>         nuke_rocket = new CMissile(Root->GetObject(25, 0), &ldata->launch, &SDesc);
> // ...
>         return;
>     }
> ```
>
> - `FireFinal` without `super` is `on_fire`: your code fires the shot in
>   place of WA. It sits in the weapon's file, so it needs no name check.
> - `Desc->SpX` and `Desc->SpY` are `fire.vx` and `fire.vy`.
> - `new CMissile(...)` from the weapon's own launch data is
>   `wa.weapons.launch{ weapon = ctx.id, ... }`.
> - `ldata->launch.explosion.damage = 0` is `missile = { explosion = { power = 0 } }`:
>   in `replace` for every shot, or in `launch` for one.

## Try it

1. Copy the new files into your `starting_guide` folder in
   `WormBench/Mods/`, or the whole folder again, and press **Refresh** in the
   WormBench Mods window.
2. Under **Bundled profiles**, pick **Scatter Bazooka — Starting guide**.
3. Start an offline game. Press F1 until the Scatter Bazooka is selected,
   charge and fire: three rockets fan out, each with a smaller blast than the
   stock Bazooka beside it.
4. The Moon Grenade is on F2, with its own icon.
5. `wkLua.log` has `[guide.scatter_bazooka] on F1, column 6: ammo 2, delay 0`.
   Change the delay of the Scatter Bazooka in **Manage Weapon Panel**, start
   a new match, and the line has the new value.

## Next

Part 3, *Options and features*, adds weapon settings that
players change in the loader (`[[components.options]]`, `ctx.option`), and
builds a feature: a component that isn't a weapon.

See also:

- [Weapon records](weapon-records.md): every record table and field.
- The [verb list](verbs.md#weapons): every `wa.weapons` function, the claim
  fields and the weapon data.
- [Moving actors](actor-movement.md): for an `on_fire` that spawns
  something WA has no missile for.
