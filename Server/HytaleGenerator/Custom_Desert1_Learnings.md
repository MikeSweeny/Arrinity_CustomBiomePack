# Custom Desert1 Learnings Log

Updated: 2026-02-11

## Purpose

This is a local memory for behavior patterns and logic flow, not a raw change history.
Use it to remember what combinations work, what fails, and why.

## Core world-gen model (from local preset files)

- `WorldStructures/*.json` (`Type: NoiseRange`) assigns biome ownership by numeric ranges on a map density.
- Biome blending happens at worldstructure level via:
  - `DefaultTransitionDistance`
  - `MaxBiomeEdgeDistance`
- Map density source is selected by `WorldStructures/.../Density/Name`.
- Biome terrain is usually built as a density equation around:
  - `BaseHeight(Base)`
  - `Inverter(YValue)`
  - plus optional noise/curve terms.
- Material assignment is independent from terrain density shape.

## Official docs seed (local references)

Seeded from:

- `/en/docs/official-documentation/worldgen/worldgen-tutorial/world-generation-concepts`
- `/en/docs/official-documentation/worldgen/worldgen-tutorial/how-to-edit-and-create-biomes`
- `/en/docs/official-documentation/worldgen/technical-hytale-generator/density`
- `/en/docs/official-documentation/worldgen/technical-hytale-generator/material-providers`

Key takeaways to reuse:

- Density fields are math fields (typically interpreted as solid > 0, empty < 0).
- Core terrain formula is usually `BaseHeight + (-Y) + modifiers`.
- `YOverride(Value=0)` is the standard way to force 2D/XZ behavior in map fields.
- `BaseHeight(Distance=true)` returns signed distance to the chosen base height in blocks.
- For baseline biome terrain, prefer `BaseHeight(Distance=true) -> CurveMapper` with simple 2-3 points and curve `Out` in `[-1, 1]`.
- Avoid the basic `BaseHeight + Inverter(YValue)` pattern during early tuning; in this project it repeatedly caused unintuitive all-solid behavior.
- `Mix` uses a gauge (0 -> A, 1 -> B) and is the standard non-binary field blending tool.
- `SmoothMin/SmoothMax` are available for softer function boundaries than hard `Min/Max`.
- `Multiplier` can act as both masking and biome "strength" modulation.
- `DistanceToBiomeEdge` returns nearest edge distance in blocks (center-symmetric behavior can appear in practice).
- MaterialProviders are layered logic; `Solidity + Queue + SimpleHorizontal/SpaceAndDepth` are the baseline stack.

## Preset map logic patterns (non-custom reference)

### `Density/Map_Tiles.json`

- Uses `YOverride(Value=0)` to force 2D XZ map logic.
- Uses `PositionsCellNoise + Mesh2D` to define biome ownership fields.
- Key controls:
  - cell size: mesh `ScaleX/Z`
  - edge softness/shape: `MaxDistance`
  - cell irregularity: `Jitter`

### `Density/Map_Default.json`

- Mixes tile map with continent and river masks.
- Uses clamps/normalizers to gate where each field dominates.
- Result: less binary biome ownership than pure tile-only graphs.

## Confirmed practical learnings

- `DistanceToBiomeEdge` is symmetric from center and can create unwanted center-out effects.
- If blend distance is too high, biomes appear to "eat" neighboring bands.
- Shore must fill empty space below water level with `Water_Source` in `MaterialProvider.Empty`.
- `Roof=170` in framework constants is a reference baseheight in this project context.
- Changing terrain noise + worldstructure ranges + map density in one pass causes unreadable regressions.
- Keep `Custom_Desert1_Map_Default.json` importing `Biome-Map` (not `Biome-Map-Tiles`) when you want canonical default continent/river behavior.
- A flat shore at `Base+10` is valid; remaining steep edge appearance can come from biome blending, not from shore material assignment.
- For edge-height shaping, `DistanceToBiomeEdge` works better as a direct height-offset curve summed with `BaseHeight(Distance=true)`; avoid a second final terrain `CurveMapper` if it causes early saturation/plateaus.
- Most intuitive shoreline equation so far: `Density = Inverter(BaseHeight(Water, Distance=true)) + EdgeHeight(distanceToEdge)`, where edge curve `Out` is directly the target shoreline height in blocks.

## Debug strategy now (agreed workflow)

1. Keep canonical map behavior from default logic via custom wrapper files:
   - `Custom_Desert1_Map_Default.json` exports `Custom_Desert1_Biome-Map` from `Biome-Map`.
   - `Custom_Desert1_Map_Tiles.json` exports `Custom_Desert1_Biome-Map-Tiles` from `Biome-Map-Tiles`.
   - Important: importing only `Biome-Map-Tiles` does NOT include default continent/river mix behavior.
   - If `Custom_Desert1_Biome-Map` imports `Biome-Map`, then full default map logic applies to tests.
2. Freeze non-playground biomes for now:
   - `Custom_Desert1_Oasis_Ocean`
   - `Custom_Desert1_Oasis_MudFlats`
   - `Custom_Desert1_Oasis_Geysers`
3. Active playground order:
   - first Shore (with Ocean only enabled)
   - then Dunes
   - then Mountains
   - then reintegrate all.
4. Isolation rule for each step:
   - Keep only currently tested biome(s) active in worldstructure.
   - For the active test phase, let the active biome own the full remaining range so behavior is easy to read.
   - Add next biome only after screenshot-confirmed success.
5. During gradient debugging, use flat target heights:
   - Shore `Base + 10`
   - Dunes `Base + 30`
   - Mountains `Base + 80`

## Visual interpretation reminders

- Sharp ring walls usually indicate ownership boundaries + big vertical difference + low transition distance.
- "Wrong floor material" often means wrong biome ownership, not necessarily wrong material provider.
- Chunk-like square patches in screenshots are often regen/update artifacts, not persistent biome math.

## Iteration protocol

- Change one subsystem only.
- Before saving, remove disconnected/floating nodes from the graph so only the active logic path remains.
- Capture top-down + in-world screenshots.
- Log whether result matched expected logic path.
- Keep successful mini-patterns for reuse; blacklist failed combinations.
