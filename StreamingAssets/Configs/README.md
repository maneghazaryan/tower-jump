# Tower Jump — configuration reference

Everything in this folder is loaded at runtime and validated before the game starts. Nothing here
is compiled in: edit a file, press Play, see the change. If a file is invalid the game **refuses to
start** and prints every problem it found, naming the file and the field.

## Where configs are loaded from

Two sources are searched in order; the first hit wins.

| Order | Location | Purpose |
| --- | --- | --- |
| 1 | `<persistentDataPath>/Configs/…` | Per-machine override. Drop a file here to retune a **built** game with no rebuild. |
| 2 | `<StreamingAssets>/Configs/…` | The shipped defaults — these files. |

On macOS `persistentDataPath` is `~/Library/Application Support/<Company>/<Product>`.

Override a single tower by copying just that one file; everything else still comes from the
shipped set.

## Files

| File | What it controls |
| --- | --- |
| `gameplay.json` | Movement feel, wall bounce, camera, run rules, streaming, platform physics, per-platform-type tuning. |
| `skins.json` | Player skins: appearance key plus stat multipliers. |
| `economy.json` | Starting coins and the continue price ladder. |
| `items.json` | Collectibles: what exists, how rare it is, what it does. |
| `towers.json` | The list of towers the lobby offers. |
| `Towers/<id>.json` | One tower (= one level). The file name **must** match the `id` field. |

---

## Tower files

A tower is described as a series of **bands** over platform indices. Index 1 is the bottom platform;
`totalPlatformCount` is the top.

```jsonc
{
  "schemaVersion": 2,          // rejected if it does not match the build
  "id": "tower_00",            // must equal the file name
  "displayName": "Stone Ascent",
  "seed": 20260911,            // same seed + same file = byte-identical tower, forever
  "totalPlatformCount": 400,
  "towerHalfWidth": 4.5,       // world units from the centre line to each wall's inner face
  "startHeight": -3.0,         // world Y of platform 1
  "fullWidthInterval": 100,    // every Nth platform spans wall to wall; 0 disables
  "spacing":               { … },  // vertical gaps
  "widthByHeight":         [ … ],  // platform width
  "spriteSetsByHeight":    [ … ],  // which art
  "scrollSpeedByHeight":   [ … ],  // difficulty curve: how fast the camera climbs
  "platformTypesByHeight": [ … ],  // which types can appear
  "completionReward": 250,     // coins for reaching the top
  "milestoneReward": 25        // coins each time a landmark floor is passed
}
```

### The band rules (enforced for every band list)

1. `fromIndex` and `toIndex` are **inclusive** and 1-based.
2. Bands must not overlap.
3. Bands must not leave gaps.
4. Together they must cover `1 … totalPlatformCount` exactly.
5. Order in the file does not matter — bands are sorted on load.

Breaking any of these produces a precise error naming the offending indices.

### Skin milestones (`skinMilestoneInterval`)

The platform theme may only change on a milestone floor — 100, 200, 300 — and the validator enforces
it:

```jsonc
"skinMilestoneInterval": 100,
"spriteSetsByHeight": [
  { "fromIndex": 1,   "toIndex": 100, "spriteSetId": "moss" },
  { "fromIndex": 101, "toIndex": 200, "spriteSetId": "stone" },
  { "fromIndex": 201, "toIndex": 350, "spriteSetId": "sand" }   // last band may run to the top
]
```

A band must begin at floor 1 or just after a milestone, and end on a milestone or at the top of the
tower. Bands that drift off the interval make the tower look like it reskins itself at random,
because from inside the game there is nothing to tell you floor 217 was special. Set the interval to
`0` to opt a tower out; omit it and it defaults to 100.

### Landmark floors (`fullWidthInterval`)

Counted from the bottom platform, so an interval of `100` makes platforms **1, 101, 201 …** span the
tower wall to wall — the floors a player counts as 0, 100, 200. Their width and position override
`widthByHeight` for those indices only; everything else is unaffected.

Because the first platform is always one of them, every seed starts the player on solid ground that
reaches both walls. Set to `0` to turn the feature off.

Changing this value does **not** re-roll the rest of the tower: the generator still draws the same
random samples for landmark platforms and simply discards the results, so the seed stays stable.

### Band payloads

| List | Fields | Notes |
| --- | --- | --- |
| `spacing.gapByHeight` | `minGap`, `maxGap` | Vertical distance to the next platform. Omit the list entirely to use the top-level `minGap`/`maxGap` everywhere. |
| `widthByHeight` | `minWidth`, `maxWidth` | Sampled per platform. `maxWidth` must fit inside `towerHalfWidth * 2`. Shrink these with height to make the tower harder. |
| `spriteSetsByHeight` | `spriteSetId` | Resolves to `Assets/Resources/Art/platform_<id>.png`. Currently `stone`, `ice`, `crystal`. |
| `scrollSpeedByHeight` | `unitsPerSecond` | How fast the camera climbs on its own **while the player is in this band**. This is the difficulty curve. `0` holds the camera still — right for the opening floors. |
| `platformTypesByHeight` | `weights: [{typeId, weight}]` | Relative weights; they do **not** need to sum to 1. |

---

## Platform physics (`gameplay.json` → `platforms`)

```jsonc
"platforms": {
  "thickness": 0.2,          // collider and sprite thickness, world units
  "oneWay": true,            // solid from the top only
  "restingTolerance": 0.06   // how far a resting body may settle into its floor
}
```

**Platforms collide from the top only.** The player jumps up through them, runs through their sides,
and lands on top. The rule, in `PlayerPlatformFilter`, is about *history*:

```
solid = (feetY at the end of the previous step) >= surfaceY - restingTolerance
```

A platform is solid only to a player who was already standing above it last step.

| Situation | Why it behaves correctly |
| --- | --- |
| Landing | Feet were above the surface → solid, the swept solve catches them even at terminal velocity. |
| Jumping up through | Feet were below → not solid. |
| Hitting a side | Level with the platform means the feet are half a body below its surface → not solid. |
| Hitting the underside | Same reason. |
| **Wedged against a wall** | Feet are below the surface, so the platform is not solid and the player falls free. Nothing can trap them. |

Reading *last* step's position rather than the current one is what makes this safe. The earlier rule
tested the current position plus vertical velocity, which meant the solver could shove the player
sideways into a platform, the test would momentarily pass, and the platform would turn solid around
them — the stuck-between-wall-and-platform bug. History is monotone: you can only be "on top of" a
platform by having actually been on top of it.

`restingTolerance` absorbs only the sub-millimetre settle of a resting contact. It is **not** a
landing window — landings are caught by the swept solve. Keep it far below half the player's height
(0.35) or a platform would count as underfoot while the player is level with its side.

---

## Surface grip — why normal platforms are not slippery

Two numbers, in two places:

```jsonc
// gameplay.json → movement
"groundOverspeedDecay": 34.0,   // how fast above-cap speed is scrubbed while grounded
"jumpChargeDuration": 0.6       // how long a landing's speed is remembered for jump height

// gameplay.json → platformTypes[]
{ "typeId": "normal", "decelerationMultiplier": 1.6, "landingSpeedRetention": 0.0 }
{ "typeId": "ice",    "decelerationMultiplier": 0.12, "landingSpeedRetention": 0.92 }
```

`landingSpeedRetention` is the slipperiness dial. On touchdown the arrival speed is split in two:

- **What the player is asking for** — a direction held at the moment of landing, worth up to
  `movement.maxSpeed` — is theirs, and no amount of grip takes it away.
- **Everything else is drift** — momentum inherited from the fall or a wall bounce rather than from
  input. `landingSpeedRetention` is the fraction of *that* which survives: `0` bites immediately
  (stone — you plant exactly where you land), `1` keeps all of it (ice — you skate on).

So on stone, landing with no key held stops the player dead in a single physics step. Land holding the
direction you are travelling and you keep running at full speed — running off an edge is then
something you did, not something the floor did to you. That distinction is the whole design: grip
removes drift, never input.

It is the **only** field in `MovementModifiers` that is not multiplicative; contributors take the
maximum, because multiplying two slippery surfaces would absurdly make the pair grippier than either.
Its identity is `0`, so anything standing on nothing at all is treated as fully grippy.

**Grip does not cost you the jump.** On every landing the arrival speed is banked into a jump charge
before any of it is scrubbed, and for `jumpChargeDuration` the next jump launches as though the speed
were still there. So a hard wall bounce into a stone platform stops the slide dead but still throws
you as high as you earned.

`decelerationMultiplier` is the separate, ordinary friction dial: how fast you stop when you *release*
the key mid-platform. In the air the above-cap excess is preserved in full, so a bounce always carries
across the gap it was earned to cross.

**Standing still spends it.** The banked charge is momentum, not a battery:

```jsonc
"movement": {
  "standStillGrace": 0.25,          // seconds of standing still before the charge is gone
  "standStillSpeedThreshold": 0.35  // speed below which you count as standing still
}
```

Come to a complete stop on the ground for `standStillGrace` and the charge is spent, so the next jump
is an ordinary one. Without this, a player could land hot, stop dead, think about it for five seconds
and still launch as though they were sprinting — which reads as a bug, because nothing on screen says
they are still carrying anything. The grace is what keeps it from punishing the normal split-second
beat between landing and pressing jump. Only *ground* time counts: the player is motionless at the
top of every arc through no choice of their own.

---

## Scale, and why the jump only just reaches

The world is sized around one relationship: **a plain jump reaches the next floor and almost nothing
more.** Everything else follows from it.

```jsonc
"movement": {
  "jumpBaseSpeed": 12.45,   // apex 1.76 units, against a widest gap of 1.52
  "gravityScale": 4.5,
  "coyoteTime": 0.12,       // grace after leaving a ledge
  "jumpBufferTime": 0.15    // grace for pressing before landing
}
```

Apex is `jumpBaseSpeed² / (2 · 9.81 · gravityScale)`. At jump scale 1.0 that is **1.76** against gaps
of **1.44–1.52** — about a sixth of the jump as landing margin. Height above one floor has to be
*earned* with horizontal speed; the base jump buys you exactly one floor.

That margin is small enough that it only works because of the two forgiveness windows:

| | What it forgives |
| --- | --- |
| `coyoteTime` | Pressing jump a moment *after* walking off a ledge — which is what everyone does, because you press when you see the edge |
| `jumpBufferTime` | Pressing a moment *before* landing, which happens constantly at speed |

**Loosen the jump margin and these can shrink; tighten it and they must grow.** They are the same
decision viewed from two sides, which is why `ShippedConfigTests.ComfortableJumpFraction` says so.

### If you change the scale

The player, the platforms and the gaps are interlocked, and the slack between them is small:

```
gap  −  thickness  −  player height  =  clearance
1.44 −    0.26     −       1.0       =     0.18
```

Shrink that to zero and the player's head is permanently inside the floor above. Four values move
together, and `ShippedConfigTests` checks every one of them against every shipped tower:

| Value | Where | Constraint |
| --- | --- | --- |
| `spacing.minGap` / `maxGap` | each tower | wide enough for player + platform + clearance |
| `platforms.thickness` | `gameplay.json` | counted twice, once at each end of the gap |
| `PlayerColliderSize` | `GameInstaller.cs` | the only one not in config, because physics needs it before any file is read |
| `run.continueSpawnOffset` | `gameplay.json` | inside the window above — see §spawn test |

---

## The wall buys height

```jsonc
"wallBounce": {
  "speedMultiplier": 0.82,     // horizontal returned, as a share of the approach — below 1 on purpose
  "speedBonus": 1.0,           // flat horizontal kicker, so slow contacts are not dead
  "verticalTransfer": 0.45,    // share of the approach speed turned into lift
  "maxVerticalBoost": 9.0,     // ceiling on what one bounce may add upward
  "ascendingVerticalBonus": 3.2
}
```

A wall in a climbing game should pay in **height**, not in sideways travel. The tower is only a few
units wide and there is nothing at the far side worth reaching, so a pure reflection makes the wall a
way to cross the room. `verticalTransfer` turns part of the approach upward instead, and
`speedMultiplier` sits below 1 to pay for it: the horizontal half is deliberately lossy, and the loss
comes back as climb.

`maxVerticalBoost` caps what the wall **adds**, not how fast the player may end up rising. A player
who arrives already climbing keeps everything they came in with — clamping the result instead would
let a wall quietly slow down a good jump.

Falling into a wall still earns nothing at all, so the lift can never rescue a botched jump.

---

## The floor collapses, and that is what kills you

The tower eats itself from the bottom. A rising line — `TowerCollapse` — sweeps upward at the tower's
own pace, and every floor it reaches is destroyed. That line, not the camera, is the threat.

Each floor gets a warning first:

| Where the floor is | What happens | Can you stand on it? |
| --- | --- | --- |
| Above the line + `shakeLeadDistance` | Nothing | Yes |
| Within `shakeLeadDistance` of the line | **Shakes** in place | **Yes** — fully solid |
| Below the line | **Breaks loose**, tumbles and fades | No |

```jsonc
"collapse": {
  "shakeLeadDistance": 2.6,   // how far ahead of the line a floor starts shaking
  "shakeDuration": 0.85,      // seconds it shakes before breaking loose, on its own clock
  "shakeAmplitude": 0.07,     // how far it slides side to side
  "shakeFrequency": 13.0,     // shakes per second
  "fallDuration": 1.6,        // seconds from breaking loose to being recycled
  "fallAcceleration": 16.0,   // how hard it drops
  "fallDrift": 0.9,           // sideways drift as it falls
  "fallSpin": 55.0,           // degrees per second of tumble
  "fadeStartFraction": 0.55,  // how far into the fall it begins to fade
  "maxDistanceBelowView": 26.0,
  "startDropDistance": 4.5    // head start below the first floor, and after every continue
}
```

**Each floor keeps its own clock.** The line only ever *starts* a floor shaking; the floor falls
`shakeDuration` later under its own count. That is what makes the collapse read as a sequence — one
floor after another at a steady rhythm — rather than a sweep that bunches up whenever the line speeds
up. Once a floor breaks loose it accelerates downward, opening a visible gap under the floor above
before it fades, so the tower is seen coming apart rather than quietly dimming.

**The shake is a promise, not a punishment.** A shaking platform is exactly as solid as any other —
you can land on it, stand on it and jump from it. That is the whole point: the warning has to arrive
before the consequence, or losing a floor is indistinguishable from a bug. Ride one down and you are
still alive; you simply need somewhere to go before it lets go.

**How fast the line climbs is per tower**, in `scrollSpeedByHeight` — that is what makes one tower
harder than another. It is sampled at the **highest floor reached**, never the current one, so falling
cannot slow the collapse and ease the pressure at the worst possible moment.

`maxDistanceBelowView` is a memory bound rather than a rule of the game: floors between the line and
the view stay alive because a falling player might land on one, and a fast climber can open that gap
indefinitely. Anything further below the screen than this is released, and the line moves up to match
so "destroyed" and "below the line" never disagree.

### The kill volume

The run is ended by a real trigger collider that sits just below the visible area and follows the
camera. Because the camera stops at the collapse line, the volume comes to rest there too — so
"falls past the last standing floor" and "reaches the kill volume" are the same moment:

```jsonc
"run": {
  "deathZoneOffset": 1.5,   // gap between the camera's bottom edge and the volume's top
  "deathZoneHeight": 20.0   // how deep the volume is
}
```

A collider rather than a height comparison, because the physics engine evaluates it with the same
sub-step accuracy as everything else the player touches — a fast fall cannot slip past it between
two frames.

`deathZoneHeight` must comfortably exceed the distance the player covers in one physics step at
`movement.maxFallSpeed` (about 0.5 units), or terminal velocity could tunnel through the trigger.
The loader rejects a value that is too shallow and tells you the number it wants.

The `Falling` run state is now only a warning that the player is below the lowest visible platform.
It has no timer: a player who claws their way back up is never punished for the dip, and the run
ends when — and only when — they reach the volume.

---

## How to extend safely

### Add a new tower

Copy `Towers/tower_00.json`, change `id` (and the file name to match) and `seed`. Point the
bootstrap at it via the `Tower Id` field on the `[Bootstrap]` object in `Assets/Scenes/Game.unity`.

### Add a new platform type

1. Add one C# file. Nothing existing is edited:

   ```csharp
   [PlatformType("sticky")]                    // the id you will write in tower JSON
   public sealed class StickyPlatformBehaviour : SurfacePlatformBehaviour
   {
       public const string TypeId = "sticky";
   }
   ```

   Deriving from `SurfacePlatformBehaviour` already gives it every tuning field below, read from
   `gameplay.json` — that is all `normal` and `ice` are. Add `ILandingResponder` on top only if it
   also needs to *react* when the player touches down.

2. Optionally add a `platformTypes` entry in `gameplay.json` to tune it. Skipping this is fine —
   unconfigured types get neutral defaults.

3. Add `{ "typeId": "sticky", "weight": 0.1 }` to a band in a tower file.

The registry discovers the class by its attribute, the validator starts accepting the id, and the
factory starts building it. There is no registry list, enum or switch statement to update.

### Add a new skin

Add a row to `skins.json` and a sprite sheet named `player_<id>` sliced with the same frame names
as the existing ones (`_idle_0`, `_run_0…5`, `_jump_0`, `_fall_0`, `_salto_0…3`). No code changes.

### Change a tower that is already live

**Do not renumber existing bands** — that changes the layout every existing seed produces, which
invalidates saved progress and leaderboards. Append new bands above the current top instead, and
raise `totalPlatformCount` to match.

### Change the file format

Bump `schemaVersion` in the DTO *and* in every file. The validator rejects unknown versions with a
clear message rather than silently misreading old data.

---

## Common validation errors

| Message | Cause |
| --- | --- |
| `'widthByHeight' has a gap: platforms 51..59 are not covered` | A band's `toIndex` and the next `fromIndex` are not consecutive. |
| `entries […] and […] overlap` | Two bands cover the same index. |
| `stops at platform 80 but the tower has 400 platforms` | The last band ends early. |
| `uses unknown typeId 'x'` | No class carries `[PlatformType("x")]`. The message lists the ids that do exist. |
| `has maxWidth 99 which does not fit inside the tower` | `maxWidth` exceeds `towerHalfWidth * 2`. |
| `file declares id 'a' but was loaded as 'b'` | The `id` field and the file name disagree. |
| `'platforms' section is required` | `gameplay.json` predates schema version 2. Add the `platforms` block shown above. |
| `'schemaVersion' is 1, expected 3` | A `gameplay.json` written before the `platforms` section existed. |
| `'schemaVersion' is 2, expected 3` | A `gameplay.json` from before the kill volume. Replace `run.deathMargin`/`run.fallGraceTime` with `run.deathZoneOffset`/`run.deathZoneHeight`, and `platforms.oneWaySurfaceArc` with `platforms.landingTolerance`/`platforms.passThroughRisingVelocity`. |
| `'run.deathZoneHeight' … is too shallow` | The volume is thin enough that a fast fall could tunnel through it. The message states the minimum. |


---

## On-screen controls (`gameplay.json` → `controls`)

```jsonc
"controls": {
  "touchMode": "auto",   // "auto" | "always" | "never"
  "minPadSize": 150.0,   // smallest a pad may be, in canvas units
  "maxPadSize": 260.0,   // largest, so it never swallows the playfield
  "edgeMargin": 46.0,    // gap from the safe-area edge
  "opacity": 0.55        // low enough to read the tower through them
}
```

Two direction pads bottom-left, jump bottom-right — the layout every player of this genre already
has in their hands, and splitting the corners is what makes "move and jump at once" a two-thumb
action rather than a stretch.

**`touchMode: "always"` is the one to reach for while developing.** It puts the pads on screen in the
editor so they can be driven with a mouse, with no device needed. `"auto"` asks Unity whether there
is a touchscreen rather than checking the platform, so a Windows tablet and a touch laptop are
handled without naming either, and a desktop build keeps a clean screen.

**Touch and keyboard are always both live.** A tablet with a keyboard case needs no mode switch, and
opposite directions on the two cancel out exactly as holding both pads does.

### How a pad is sized

A share of the canvas's **shorter edge**, then clamped by `minPadSize`/`maxPadSize`, then capped so
two direction pads plus their margin cannot reach across into the jump pad.

The shorter edge is the one that constrains a thumb in both orientations: sizing from height alone
gives a phone in landscape pads that swallow the playfield, and from width alone a tall phone pads
too small to hit. Rotating a device therefore does not change the pad size.

Note these are **canvas units, not device pixels** — the scaler normalises resolution away before the
layout sees it, so a 640px phone and a 1440px one both present a canvas about 1080 units across.

Where the limits disagree, **fitting beats comfort**: a pad below `minPadSize` is awkward, but two
pads the player cannot tell apart are worse.

---

## The camera

```jsonc
"camera": {
  "orthographicSize": 7.0,   // minimum half-height; widened if the screen is too narrow
  "verticalOffset": 1.5,     // the player's anchor line, below the camera centre
  "smoothTime": 0.12,        // easing for handovers and small corrections
  "maxFollowLag": 0.9        // hard cap on how far the camera may trail its target
}
```

The camera is a **window, not a threat**. It used to be both — scrolling upward at a fixed rate and
destroying whatever it left behind — and those two jobs want opposite things, which is why it could
never be allowed to follow a falling player. The pressure now lives in the collapse line, leaving one
rule:

```
targetY = max(player.y + verticalOffset, collapseLine + halfHeight)
```

| Situation | Result |
| --- | --- |
| Player climbs | The camera follows them up. |
| Player falls | The camera follows them **down**. |
| Player falls past the last standing floor | The floor term wins, the camera stops, and the player drops out of the bottom of the screen. That is the game over. |
| Player dawdles | The rising floor pushes the camera up from beneath them, which is what puts the shaking platforms at the bottom edge where they can be seen. |

The floor term is the only limit on how far down it will look, and it is there for a reason: below
the line there is nothing to stand on and nothing to see, and showing it would hide the shaking
floors the player is being warned about.

`maxFollowLag` is the important knob. Smoothing alone lags a moving target in proportion to its
speed, so a hard jump would walk the player up the screen before the camera caught up. The cap bounds
that drift in both directions. Lower is tighter and more rigid, higher is looser and floatier.

## Items (`items.json`)

```jsonc
{
  "schemaVersion": 1,
  "spawnChancePerPlatform": 0.035,  // master rarity dial: 3.5% of eligible floors carry anything
  "minimumFloor": 12,               // opening floors never carry items
  "items": [
    { "id": "coin",   "spriteId": "item_coin",   "weight": 0.9,
      "fromFloor": 0, "toFloor": 0, "coinReward": 5, "liftFloors": 0,  "liftSpeed": 0.0 },
    { "id": "rocket", "spriteId": "item_rocket", "weight": 0.1,
      "fromFloor": 25, "toFloor": 0, "coinReward": 0, "liftFloors": 20, "liftSpeed": 22.0 }
  ]
}
```

Two independent dials: `spawnChancePerPlatform` decides **whether** a floor carries anything at all,
and `weight` decides **which** item it is once that roll has succeeded. `fromFloor`/`toFloor` bound
where an item may appear (`0` means unbounded at that end), so different floor ranges can hold
different items. Placement is planned once from the tower's seed, on its own random stream — so a
seed always puts the rocket on the same floor, and retuning rarity never reshuffles the platforms.

Landmark floors never carry items: a rocket sitting on one would routinely skip the next landmark.

The rocket's distance is expressed in **floors**, not world units, so "twenty floors" means the same
thing in every tower regardless of its spacing.

---

## Economy (`economy.json`)

```jsonc
{
  "schemaVersion": 1,
  "startingCoins": 50,          // one-time grant so the first continue offer is usable
  "continueCosts": [10, 20, 50],// price of the 1st, 2nd, 3rd continue within a run
  "costGrowthBeyondList": 2.0   // each further continue costs this much more than the last
}
```

Continue prices escalate so each one is a harder decision than the last; the ladder resets when a new
run starts. Once the list runs out the last price keeps multiplying, so the curve never needs an
unbounded list.

Coins are earned by climbing: `milestoneReward` each time a landmark floor is passed (platform 1 is
excluded — you start on it), and `completionReward` for reaching the top. Both live in the tower
file, so each tower prices its own difficulty.

The balance is stored in `<persistentDataPath>/player.json`. Delete that file to simulate a brand new
player.

### Ways to continue

The game-over popup offers three, but only one is implemented:

| Option | Status |
| --- | --- |
| Pay coins | Works. |
| Watch a rewarded ad | Shown disabled — no ad SDK is integrated. |
| Buy coins | Shown disabled — no store is integrated. |

The unimplemented ones are shown greyed out with the reason on the button rather than hidden, so the
flow stays honest about what is missing. Integrating either means writing one `IContinueProvider` and
binding it in `GameInstaller`; no gameplay file changes.

---

## Skins (`skins.json`)

```jsonc
{
  "schemaVersion": 2,
  "defaultSkinId": "default",
  "skins": [
    { "id": "default",   "displayName": "Rookie",    "description": "Balanced…", "unlockCost": 0,
      "spriteSetId": "player_default", "maxSpeedMultiplier": 1.0, "accelerationMultiplier": 1.0,
      "jumpMultiplier": 1.0, "bounceMultiplier": 1.0, "airControlMultiplier": 1.0 }
  ]
}
```

`unlockCost` of `0` means the skin is free and owned by everyone from the first run. Anything higher
is bought once with coins and then owned forever.

**Free skins are not written into the save.** Owning a free skin is a fact about the catalog, not
about the player — recording it would mean a save written before a skin became free kept it locked.
The save holds only what was actually bought, plus which skin is equipped.

The lobby's **SKINS** button opens the picker. It lists every skin in the catalog, not just the ones
you own: a menu that shows only what you already have tells you nothing about what the coins are for.
Locked rows are dimmed, show their price, and render the character as a silhouette — recognisable
enough to want, plainly not yours.

A selection survives leaving the menu, leaving the lobby, and starting a run. If the saved id names a
skin that no longer exists or is not owned — a removed skin, a hand-edited save — it falls back to
the catalog default rather than erroring.

---

## The lobby (`towers.json`)

```jsonc
{
  "schemaVersion": 1,
  "towers": [
    { "id": "tower_00", "displayName": "Stone Ascent", "description": "400 floors…" }
  ]
}
```

A manifest rather than a directory scan: StreamingAssets cannot be enumerated on every platform, and
an explicit list lets a work-in-progress tower exist on disk without being offered yet.

Adding a tower is two edits — a `Towers/<id>.json` file and a row here. The chosen tower is passed to
the game scene through `player.json`, so opening `Game.unity` directly still works and replays
whatever was picked last.


---

## How do I…

### …add a new platform, wall or background skin?

1. Add a palette entry to `PLATFORM_MATERIALS` in `Tools/ArtGen/generate_art.py`, named
   `platform_<skin>`, and run `python3 Tools/ArtGen/generate_art.py`.
2. Reference it as `"<skin>"` in a tower's `spriteSetsByHeight` band.

Wall and background themes work the same way, via `WALL_THEMES` and `BACKGROUND_THEMES`; a tower
picks them by name in its `environment` block. New art is found automatically, so there is no list to
add it to.

No gameplay code changes: a skin is purely visual, and every normal platform behaves identically
whichever skin it is wearing.

### …assign a skin to a new 100-floor range?

Add a band to that tower's `spriteSetsByHeight`. To start a new skin at floor 501, extend
`totalPlatformCount` past 500 and append
`{ "fromIndex": 501, "toIndex": 600, "spriteSetId": "yourskin" }`. Bands must tile
`1…totalPlatformCount` with no gaps or overlaps; the loader says exactly which indices are missing if
they do not.

### …change the probability of normal / slippery / trampoline?

Edit `platformTypesByHeight` in the tower file. Weights are relative and need not sum to 1, so adding
a type does not mean rebalancing every other number. A test pins normal at ≥80% of every tower, so if
you push specials too far the suite tells you.

### …add a new item?

One class plus one config row:

```csharp
[ItemType("magnet")]
public sealed class MagnetItemBehaviour : IItemBehaviour
{
    public void OnSpawn(in ItemDefinition definition) { }
    public void OnCollected(IItemContext context) => context.AwardCoins(10);
}
```

Then add `{ "id": "magnet", "spriteId": "item_magnet", "weight": 0.2, … }` to `items.json`. The
registry finds the class by its attribute and the loader starts accepting the id. Nothing existing is
edited. The loader rejects an id with no behaviour, and names the ids that do exist.

### …change item rarity?

`spawnChancePerPlatform` for items overall; an item's `weight` for its share of those. Set a weight
to `0` to disable an item without deleting its row.

### …change the rocket from 20 floors to another distance?

`items.json` → the rocket's `liftFloors`. `liftSpeed` controls how fast the ride is.

### …change the coin reward?

`items.json` → the coin's `coinReward`.

### …add a new tower?

Copy a file in `Configs/Towers/`, change `id` (and the filename to match) and `seed`, then add a row
to `towers.json` so the lobby offers it. Give it an `environment` block to restyle its walls and
backdrop.

### …change what a skin costs, or give one away free?

`skins.json` → that skin's `unlockCost`. Zero makes it free for everyone immediately, including
players whose saves predate the change.

### …add a new player skin?

Add a palette to `PLAYER_SKINS` in `Tools/ArtGen/generate_art.py`, regenerate, and add a row to
`skins.json`. That is all — art under `Resources/Art/` is discovered automatically, so no C# changes.
Stats are pure data; there is no per-skin code path anywhere.

### …change platform friction or behaviour?

`gameplay.json` → `platformTypes[]` for an existing type's numbers. `landingSpeedRetention` for how
much a landing slides, `decelerationMultiplier` for how fast you stop when you release the key.

For a new surface that only needs different numbers, derive from `SurfacePlatformBehaviour` and add
nothing — that is all `normal` and `ice` are. For genuinely new behaviour, add a class with
`[PlatformType("id")]` implementing `ILandingResponder` (reacts to landings) and/or
`ISurfaceModifier` (changes how you move on it).

### …tune the camera?

`gameplay.json` → `camera`. Start with `maxFollowLag` for how tightly it holds the player, and
`smoothTime` for how soft the handovers feel. Per-tower pacing lives in each tower's
`scrollSpeedByHeight`.

### …change default gameplay values?

`gameplay.json` is the single home for everything the motor, camera, streamer and run flow use. It is
validated on load: bad values are rejected by name with the reason, and the game refuses to start
rather than limping along with defaults.
