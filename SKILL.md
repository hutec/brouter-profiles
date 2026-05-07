---
name: brouter-profiles
description: >
  Expert guide for creating and editing BRouter routing profiles (.brf files).
  Use this skill whenever the user wants to create a new BRouter profile, modify an
  existing one, understand BRouter profile syntax, tune elevation/traffic/surface
  parameters, or debug a profile. Trigger even for casual mentions like "tweak my
  brouter profile", "why does brouter avoid this road", "write a profile for X riding
  style", or "what does costfactor mean in BRouter".
---

# BRouter Profile Development Guide

BRouter profiles are scripts that define how the router calculates the cost of traversing
each road segment. The profile with the **lowest total cost path** wins. All costs are in
"equivalent meters" — the costfactor multiplies the physical segment length.

## Profile Structure

Every `.brf` profile has three context sections evaluated in order:

```
---context:global   # constants, elevation params, kinematic model
---context:way      # per-segment cost calculation (most logic lives here)
---context:node     # barrier / traffic light costs at intersections
```

### Syntax: Polish (prefix) notation

```brf
assign <variable> <expression>

# operators:
switch <bool> <if-true> <if-false>      # ternary
if <bool> then <val> else <val>         # same, more readable
or  <bool> <bool>                        # logical or (only 2 operands)
and <bool> <bool>                        # logical and
not <bool>
add <num> <num>   sub   multiply   max   min
equal <num> <num>   greater   lesser
highway=primary|secondary               # pipe = OR match on one tag
```

Expressions nest recursively. Parentheses are optional but enforce boundaries.
`assign` can only appear at top-level.

---

## Key Predefined Variables

### Global context
| Variable | Purpose | Default |
|---|---|---|
| `validForBikes` | Mark as bike profile | — |
| `uphillcost` | Elevation cost per metre of climb per km | 0 |
| `uphillcutoff` | Gradient % below which climb is free | 1.5 |
| `downhillcost` | Elevation cost per metre of descent per km | 0 |
| `downhillcutoff` | Gradient % below which descent is free | 1.5 |
| `elevationpenaltybuffer` | Buffer (m) before elevation converts to cost | 5 |
| `elevationmaxbuffer` | Buffer max (m); above this, full conversion | 10 |
| `elevationbufferreduce` | Rate at which buffer converts to cost (slope %) | 0 |
| `pass1coefficient` | Heuristic coefficient for routing pass 1 | — |
| `turnInstructionMode` | 0=none 1=auto 2=locus 3=osmand | 1 |
| `considerTurnRestrictions` | Honour OSM turn restrictions | true |
| `processUnusedTags` | Show all OSM tags in data tab (debug) | false |
| `totalMass` | Bike+rider kg (kinematic model) | — |
| `maxSpeed` | km/h cap (kinematic model) | — |
| `S_C_x` | Drag coeff × area × ½ρ (kinematic model) | — |
| `C_r` | Rolling resistance coefficient | — |
| `bikerPower` | Average watts (kinematic model) | — |

### Way context (assigned per segment)
| Variable | Purpose |
|---|---|
| `costfactor` | **Required ≥ 1.** Main cost multiplier on segment length. ≥10000 = forbidden. |
| `uphillcostfactor` | Replaces `costfactor` in elevation cost calc on uphills (optional) |
| `downhillcostfactor` | Replaces `costfactor` in elevation cost calc on downhills (optional) |
| `turncost` | Cost (m) for a 90° turn at junctions |
| `initialcost` | One-time cost added when `initialclassifier` changes (e.g. ferries) |
| `initialclassifier` | Value change triggers `initialcost` |
| `priorityclassifier` | Road importance for voice hint generation (0–30) |
| `nodeaccessgranted` | Set true on cycleroutes to bypass node barriers |

### Node context
| Variable | Purpose |
|---|---|
| `initialcost` | Cost added when traversing this node (barriers, signals) |

---

## Elevation Buffer Mechanics

The buffer absorbs small elevation changes so noise doesn't inflate cost.

1. Each segment: cutoff eats `10 × cutoff% × length_km` metres of elevation.
2. Remainder goes into the buffer.
3. When buffer > `elevationpenaltybuffer`, partial conversion begins at rate `elevationbufferreduce`.
4. When buffer > `elevationmaxbuffer`, full conversion: `elevation_m × uphillcost` added to cost.
5. `uphillcostfactor` / `downhillcostfactor` replace `costfactor` in this calculation.

**Practical defaults for a moderate elevation profile:**
```
uphillcost = 60 / downhillcost = 40
uphillcutoff = downhillcutoff = 1.5
elevationpenaltybuffer = 5
elevationmaxbuffer = 10
elevationbufferreduce = 0.5
```

---

## Highway Cost Reference (typical bike profiles)

| `highway=` | Quiet/low-traffic | Moderate avoidance | Strong avoidance |
|---|---|---|---|
| motorway | 10000 | 10000 | 10000 |
| trunk | 4–8 | 6–10 | 10000 |
| primary | 1.5–2 | 2.5–3.5 | 3.5–6 |
| secondary | 1.2–1.5 | 1.8–2.5 | 2.5–4 |
| tertiary | 1.0–1.2 | 1.3–1.6 | 1.8–3 |
| unclassified | 1.0–1.2 | 1.0–1.3 | 1.0–1.5 |
| residential | 1.0–1.3 | 1.1–1.4 | 1.2–1.6 |
| cycleway | 1.0–1.2 | 1.0–1.2 | 1.0 |
| track grade1 | 1.0–1.1 | 1.0–1.2 | 1.0–1.3 |
| track grade2 | 1.0–1.3 | 1.1–1.5 | 1.2–1.8 |
| track grade3 | 1.2–2.0 | 1.3–2.5 | 1.5–3.5 |
| track grade4+ | 2.0–5.0 | 2.5+ | 4.0+ |

Costfactor **must be ≥ 1** — values < 1 break routing.
Values **≥ 10000** = completely forbidden (segment deleted from graph).
Value **9999** = excluded from routing but used for voice hints.

---

## Surface Quality Pattern

```brf
assign surfacepenalty =
  switch surface=asphalt                      1.0
  switch surface=concrete|paving_stones       1.05
  switch surface=fine_gravel|compacted        1.1
  switch surface=gravel                       1.2
  switch surface=unpaved                      1.35
  switch surface=ground|grass|dirt|earth|mud  2.0
  switch surface=cobblestone|sett             1.4
  1.0  # default (unknown surface = neutral)

assign tracktypepenalty =
  switch tracktype=grade1  1.0
  switch tracktype=grade2  1.1
  switch tracktype=grade3  1.3
  switch tracktype=grade4  1.8
  switch tracktype=grade5  2.5
  switch highway=track     1.3  # unknown tracktype
  1.0

# Combine for tracks:  multiply surfacepenalty tracktypepenalty
```

---

## Traffic Penalty Pattern

```brf
assign trafficpenalty =
  if not consider_traffic then 0
  else if highway=primary|primary_link then
    if      estimated_traffic_class=6|7 then 1.6
    else if estimated_traffic_class=5   then 1.0
    else if estimated_traffic_class=4   then 0.6
    else if estimated_traffic_class=3   then 0.3
    else 0
  else if highway=secondary|secondary_link then
    if      estimated_traffic_class=5|6|7 then 1.2
    else if estimated_traffic_class=4     then 0.7
    else if estimated_traffic_class=3     then 0.3
    else 0
  else if highway=tertiary|tertiary_link then
    if      estimated_traffic_class=4|5|6|7 then 0.8
    else if estimated_traffic_class=3       then 0.3
    else 0
  else 0
```

---

## Access Pattern (bike + foot)

```brf
assign defaultaccess =
  if access= then not motorroad=yes
  else if access=private|no then false
  else true

assign bikeaccess =
  if bicycle= then
    ( if bicycle_road=yes then true
      else if vehicle= then ( if highway=footway then false else defaultaccess )
      else not vehicle=private|no )
  else not bicycle=private|no|dismount|use_sidepath

assign footaccess =
  if bicycle=dismount then true
  else if foot= then defaultaccess
  else not foot=private|no|use_sidepath

assign accesspenalty =
  if bikeaccess then 0
  else if footaccess then 4
  else if any_cycleroute then 15
  else 10000
```

---

## Uphill/Downhill Costfactor for Surface Differentiation

Use `uphillcostfactor` and `downhillcostfactor` to make the router prefer or avoid
specific surfaces depending on direction (e.g. gravel uphill = good traction, preferred;
gravel downhill = slippery, avoided):

```brf
assign is_gravel_surface =
  or surface=gravel|fine_gravel|compacted|unpaved
  and highway=track tracktype=grade2|grade3

# Gravel uphills: lower elevation cost → preferred over paved alternatives
assign uphillcostfactor   = if is_gravel_surface then multiply costfactor 0.85 else costfactor
# Gravel downhills: higher elevation cost → prefer paved on descents
assign downhillcostfactor = if is_gravel_surface then multiply costfactor 1.40 else costfactor
```

These **replace** `costfactor` in elevation calculations only. The base segment cost
(costfactor) is unchanged.

---

## Costfactor Skeleton (bike profile)

```brf
assign costfactor =
  if ( and highway= not route=ferry )              then 10000
  else if ( highway=motorway|motorway_link )        then 10000
  else if ( motorroad=yes )                         then 10000
  else if ( highway=proposed|abandoned|construction ) then 10000
  else min 9999
  add max onewaypenalty accesspenalty
  add trafficpenalty
  if ( highway=steps )             then ( if allow_steps then 50 else 10000 )
  else if ( route=ferry )          then ( if allow_ferries then 5.67 else 10000 )
  else if ( highway=trunk|trunk_link ) then ( if any_cycleroute then 2.0 else 8.0 )
  else if ( highway=primary|primary_link )    then multiply surfacepenalty 2.5
  else if ( highway=secondary|secondary_link ) then multiply surfacepenalty 1.8
  else if ( highway=tertiary|tertiary_link )   then multiply surfacepenalty 1.35
  else if ( highway=unclassified )             then multiply surfacepenalty 1.1
  else if ( highway=cycleway )                 then multiply surfacepenalty 1.0
  else if ( highway=track|road )               then multiply surfacepenalty tracktypepenalty
  else if ( highway=path )                     then ...
  else 10000
```

---

## Node Context Skeleton

```brf
---context:node

assign bikeaccess
  or nodeaccessgranted=yes
  switch bicycle=
    switch vehicle=
      defaultaccess
      not or vehicle=private vehicle=no
    not or or bicycle=private bicycle=no bicycle=dismount

assign barrierpenalty =
  switch barrier=             0
  switch barrier=block|bollard  25
  switch barrier=gate|swing_gate 50
  switch barrier=cycle_barrier   87
  139  # default unknown barrier

assign initialcost =
  add barrierpenalty
  switch and nobikeaccess nofootaccess   1000000
  switch railway=level_crossing          300
  switch nobikeaccess                    200
  switch railway=crossing                35
  switch not traffic_calming=            25
  switch highway=traffic_signals|crossing 20
  0
```

---

## Kinematic Model Parameters by Bike Type

| Bike type | totalMass | maxSpeed | S_C_x | C_r | bikerPower |
|---|---|---|---|---|---|
| Racing road bike | 80 | 45 | 0.225 | 0.004 | 200 |
| Gravel/allroad | 85 | 32 | 0.275 | 0.006 | 150 |
| Trekking/touring | 100 | 28 | 0.30 | 0.010 | 100 |
| MTB | 95 | 25 | 0.35 | 0.012 | 120 |

---

## Debugging Tips

1. Set `assign processUnusedTags = true` to see all OSM tags in BRouter-web Data tab.
2. Use [brouter.de/brouter-web](https://brouter.de/brouter-web) to upload and test profiles interactively.
3. Inspect per-segment cost in the Data tab to diagnose unexpected routing.
4. A very high average costfactor (≫1) will cause slow routing — keep typical roads near 1.
5. Profiles with `costfactor < 1` on any way will produce wrong results.

## Reference

See also: `references/lookups.md` for the full OSM tag/value lookup table used by BRouter.
