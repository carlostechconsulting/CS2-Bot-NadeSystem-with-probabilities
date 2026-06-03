# Implementation Prompt: Add `weight` probability to NadeSystem

## Role
You are a CounterStrikeSharp / C# plugin developer. Implement an optional
per-grenade `weight` field in the **NadeSystem** plugin so bots throw a given
lineup only some of the time, making utility usage feel human instead of
deterministic. The change must be fully backward-compatible with existing JSON
files that have no `weight` field.

## Background / how the plugin works (read first)
- Each grenade JSON entry is deserialized into `GrenadeData` via
  `System.Text.Json`. Unknown JSON properties are **ignored by default**, so
  adding a new optional property is safe and old files keep working.
- A bot throws a lineup when it physically walks within `ZoneRadius` of the
  grenade's recorded **origin** (`projectilePosition`) — see `CheckBotZones()`.
  After the zone hit there are gates: cooldown, facing direction, live-enemy,
  smoke overlap, and the situational/economy checks in
  `PassesSituationalCheck` / `TryReplay`.
- Realism today is controlled by `bot_nades` mode (`off | normal | more | max`)
  plus economy/round-limit gates. `weight` adds a finer per-lineup dial **on
  top** of all that.

## Goal
Give every grenade an independent probability `weight` in `[0,1]` that it is
actually thrown when its trigger conditions are met. `weight` is NOT
normalized; the leftover probability is "throw nothing this time", which is the
realism we want. Missing/absent `weight` must behave exactly as today
(i.e. default `1.0` = always eligible).

## Requirements

### 1. Data model
In `GrenadeData`, add an optional property:

```csharp
// Probability (0..1) this lineup is actually thrown when triggered.
// Absent in JSON => 1.0 (always eligible, current behavior).
[JsonPropertyName("weight")]
public float Weight { get; set; } = 1f;
```

### 2. Sanitize on load
In `LoadDb()`, inside the per-entry loop (next to the existing
`entry.Description ??= "";` line), clamp the value so bad data can't disable a
grenade forever or exceed 1:

```csharp
if (entry.Weight <= 0f)      entry.Weight = 1f;   // treat 0/negative/missing-as-0 as "always"
else if (entry.Weight > 1f)  entry.Weight = 1f;
```

> Rationale: `float` defaults to `0f` only if JSON explicitly sets it to 0;
> a truly absent field keeps the property initializer `1f`. Mapping `<=0` to
> `1f` makes a literal `"weight": 0` mean "use default" rather than "never
> throw", which prevents accidentally muting the whole dataset (the exact
> failure we want to avoid).

### 3. The probability gate
Add a single helper:

```csharp
// Returns false when this lineup should be skipped this attempt due to its weight.
private static bool PassesWeightRoll(GrenadeData g)
    => g.Weight >= 1f || Random.Shared.NextDouble() < g.Weight;
```

Place the roll **after** the spatial/cooldown checks but **before** committing a
throw, in `CheckBotZones()`. Put it right after the existing
`if (IsOnCooldown(g.Id)) continue;` line, before the smoke-overlap / direction /
mode logic:

```csharp
if (IsOnCooldown(g.Id)) continue;

// Per-lineup probability dial (weight). Skip this attempt if the roll fails.
if (!PassesWeightRoll(g)) continue;
```

### 4. Decoy path
The decoy branch returns early before the cooldown line. Apply the same gate
inside the decoy block, after its cooldown check:

```csharp
if (gtype == "decoy")
{
    if (IsOnCooldown(g.Id)) continue;
    if (!PassesWeightRoll(g)) continue;          // <-- add
    if (dx * dx + dy * dy > 200f * 200f) continue;
    ...
}
```

### 5. Retaliation / special spawns (decision needed)
`HandleRetaliationHE`, `HandleMolotovEscape`, defuse/plant smokes, and
`TrySpawnInstantGrenade` are **reactive** utilities, not walk-into-zone throws.
Default: **do NOT** apply `weight` there — those are situational and already
probability-gated. Only gate the proactive `CheckBotZones` path (steps 3–4).
(If you want weight to also damp retaliation lineups, add `PassesWeightRoll(g)`
to the `candidates` filter in `HandleRetaliationHE`, but keep that opt-in.)

## Important: do NOT break the trigger density
`weight` is a *frequency* dial, not a substitute for having enough trigger
pads. Bots only throw where a recorded **origin** exists, so keep the dataset
dense at common positions. Lowering weights on a thin dataset will silence bots.
Recommended: ship the full/near-full dataset and tune realism via `weight`
(e.g. 0.2–0.6 on common lineups) + `bot_nades normal`, rather than by deleting
entries.

## JSON format
Files keep the original schema; `weight` is appended and optional:

```json
{
  "id": "….",
  "mapName": "de_dust2",
  "grenadeType": "smoke",
  "projectilePosition": { "x": …, "y": …, "z": … },
  "projectileVelocity": { "x": …, "y": …, "z": … },
  "landingPosition":    { "x": …, "y": …, "z": … },
  "description": "smoke S->N long-range",
  "weight": 0.35
}
```

## Acceptance criteria
1. Existing JSON files **without** `weight` load and behave exactly as before
   (every eligible lineup is throwable; no regression).
2. A lineup with `"weight": 0.0` is treated as `1.0` (never silently muted).
3. A lineup with `"weight": 0.3` is thrown on roughly 30% of the attempts where
   it would otherwise fire (verify over many trigger events; allow variance).
4. `weight > 1` is clamped to `1`.
5. No change to reactive utility (retaliation / molotov-escape / defuse / plant)
   unless explicitly opted in.
6. Plugin compiles with no new warnings; `bot_nades off/normal/more/max` all
   still function.

## Manual test
- Load a map, set `bot_nades max` (fewest extra gates), edit one nearby lineup
  to `"weight": 0.0` → confirm it's clamped and still throws.
- Set the same lineup to `"weight": 0.25`, repeatedly walk a bot through its
  origin → confirm it throws ~1 in 4 triggers (watch the
  `[NadeSystem] Replayed …` console lines).
- Remove the field entirely → confirm it throws every trigger.

## Out of scope
- No normalization across lineups, no "exactly N per round" budgeting (the
  per-round counters in `normal` mode already cover that).
- No changes to the position/zone trigger model.
```
