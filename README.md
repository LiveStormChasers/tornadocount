# tornadocount

Live NWS tornado count dashboard — confirmed surveys, preliminary reports, warnings, and max EF ratings by state, on a single interactive US map.

Fetches from four independent sources on demand and reconciles them into one view. Built for LiveStormChasers.com to track active seasons and export clean map graphics for publication.

Copyright (c) 2024 Live Storm Chasers LLC. All rights reserved.
Source-visible, not open source. See LICENSE — using this code requires written permission.

---

## Why

The problem is that no single NWS or SPC endpoint gives the full picture. DAT surveys are the authoritative confirmed count but lag by 24–72 hours. SPC filtered reports are faster but include duplicates. IEM LSR gives near-realtime preliminary reports. Warnings come from IEM VTEC. Combining them in one view means running four fetches, reconciling state assignments, and applying manual overrides where the source data is wrong.

The alternative — pointing users at four separate pages — doesn't produce a publishable graphic.

## Files

```
index.html    everything: data fetching, map rendering, state tables, export
```

No build step. No dependencies to install. The HTML file is the deployment artifact.

## Data sources

| Source | What it provides | Latency |
|---|---|---|
| NWS DAT ArcGIS FeatureServer/1 | Confirmed tornado surveys, EF ratings, path length | 24–72 h |
| IEM LSR GeoJSON (type=T) | Preliminary tornado reports | Near-realtime |
| IEM VTEC watchwarn / parseVTECcsv | Tornado warnings and watches | Near-realtime |
| NWS Norman OUN worker | Oklahoma statewide confirmed count | Near-realtime |
| SPC `_rpts_filtered_torn.csv` | Filtered preliminary reports | Same day |
| SPC `1950-2025_actual_tornadoes.csv` | Historical EF-era max ratings by state | Static |

The OUN worker is a separate Cloudflare Worker that scrapes the NWS Norman public count. Oklahoma's DAT total tends to lag behind what OUN publishes; the worker fills that gap and its result overrides the DAT count for OK.

## Map

Four modes on one SVG canvas, all using the same projection and coordinate space:

- **SPC Reports** — filtered preliminary counts by state, SPC color scale
- **DAT Confirmed** — NWS survey counts, OUN override applied for OK
- **Warnings** — IEM VTEC tornado warning counts, IEM spectral scale
- **Max EF (Historical)** — highest EF rating recorded per state since 2007, merged with the current year's DAT surveys

### Projection

`d3.geoAlbersUsa().scale(W * 1.25).translate([W/2, H/2])` rendered into a fixed **960 × 620** SVG coordinate space. All label coordinates are in that space regardless of how the SVG is scaled by CSS.

### State label placement — two systems

**Regular states** use D3's computed centroid, shifted by `CENTROID_FIX` for eight states where the centroid lands in water or a panhandle (LA, MI, FL, CA, ID, MS, SD, VA). Font size adapts to the state's bounding box using Anton font's measured character width of approximately 0.45× font size. Text is rendered at `centroid_y + adaptFs * 0.38` — that offset is the Anton cap-height correction; remove it and every label reads too high.

**Nine small NE states** (NH, VT, MA, RI, CT, NJ, DE, MD, DC) use hardcoded callout positions in `ABS_LABELS`: an anchor dot on the state polygon and a label floated to the right, connected by a stroke+fill line pair. The anchor coordinates are fixed in source. The label coordinates are user-draggable and persist to `localStorage('tornadomap_callouts')`. A stale-detection check on boot clears the cache if the saved NH position looks like it came from an old coordinate system.

### isSmall threshold

States whose bounding box is below `sw < fontSize * 1.8 || sh < fontSize * 0.9` fall through to a short callout line rather than a centered label. The multipliers are tighter than the obvious values on purpose — loosening them past about 2.0× catches Indiana, which is narrow but large enough to label normally.

### Known SPC data errors, corrected in code

Two states have wrong max EF ratings in the SPC historical CSV and are clamped:

- **HI**: SPC shows EF-0; NWS Honolulu rated the 2009 Kapolei tornado EF-1. Clamped to EF-1.
- **WV**: SPC shows EF-2; the Sep 2010 Wood/Wirt and Mar 2012 Lincoln/Mingo tornadoes were confirmed EF-3. Clamped to EF-3.

All other state max ratings have been verified against Wikipedia's state tornado lists and NWS survey records.

## State resolution

DAT events carry a WFO field, not a state. Resolution goes through three tiers:

1. Single-state WFOs resolve directly via a lookup table (`WFO_ST`).
2. Multi-state WFOs (e.g. IWX covers IN, OH, MI) run a `d3.geoContains` point-in-polygon check against the TopoJSON geometry to assign the correct state.
3. The OUN override is re-applied after geo-correction rebuilds the state map.

The geo-correction pass triggers a second render and returns early from the first to avoid a double-label artifact. `_geoStateCorrected` is set after the first correction so it only runs once.

## Export

**Export PNG** renders the SVG to a 3× resolution canvas. Before exporting, state counts can be manually overridden via the Override panel (amber button): enter a state abbreviation and a replacement count, apply, then export. Overrides affect SPC, DAT, and Warnings modes; Max EF mode expects 0–5 ratings, not counts.

**Copy for MapChart** copies the current state data formatted for mapchart.net.

## Things that cost time

1. **Projection scale.** The correct formula is `scale(W * 1.25)`. Using `scale(W * 1280/960)` gives a different centroid — off by ~19 px in X for Virginia. The two expressions look equivalent but 1.25 ≠ 1.333.

2. **Anton baseline offset.** `addLabel` places text at `y + adaptFs * 0.38`. Any label positioning that doesn't account for this offset will appear ~10 px too high. The 0.38 factor is specific to Anton at the sizes used here; do not assume it transfers to other fonts.

3. **isSmall false positive.** Increasing the base font size by 2 px raises the `isSmall` threshold enough to catch Indiana. Indiana's bounding box width is ~55 px in SVG space; the threshold needs to stay below that. Current multiplier is 1.8×.

4. **Double-label ghost.** When geo-correction rebuilds `stMap` and calls `renderSPCMap()` recursively, the first render must `return` immediately after the recursive call. Without the return, the first render continues and places a second set of labels over the corrected ones.

5. **OUN override lost on geo-correction.** `stMap` is rebuilt from scratch during geo-correction. The OUN count for OK must be re-applied after the rebuild, not before.

6. **JAN WFO missing from lookup.** JAN (Jackson MS) was absent from `WFO_ST`, leaving ~47 Mississippi events with an undefined state. Adding `JAN:'MS'` fixed it.

7. **SVG width attribute.** The SVG carries `style="width:100%"`, which overrides any `width` attribute D3 sets. When reading `W` from the element, `+"100%"` is `NaN`; the code falls back to 960 correctly, but the displayed width will differ from 960. Label coordinates are in SVG space (960 units wide), not display pixels.

## Not done

- **Alaska and Hawaii in non-EF modes.** AK and HI render correctly on the Max EF map (inset positions from d3's AlbersUSA) but have not been verified for label placement in SPC, DAT, and Warnings modes during active events.
- **Multi-day custom date ranges.** The date picker accepts custom ranges but fetching across long spans has not been tested for DAT — the ArcGIS endpoint has an undocumented record limit that may silently truncate results.
- **OUN worker failover.** If the worker is down, the Oklahoma count shows as DAT only with no visible error. There is no retry or fallback.
- **localStorage collision.** The callout position cache key is unnamespaced. Running two instances of this dashboard in the same browser origin would share callout state.

## Attribution

- **NWS Damage Assessment Toolkit** — NOAA/NWS. Public domain.
- **IEM Local Storm Reports and VTEC data** — Iowa State University Iowa Environmental Mesonet. Free for all use.
- **SPC storm reports and historical tornado database** — NOAA Storm Prediction Center. Public domain.
- **US Atlas TopoJSON** — Mike Bostock. ISC licence.
- **D3.js** — Mike Bostock et al. ISC licence.

## Licence

Copyright (c) 2024 Live Storm Chasers LLC. All rights reserved.
Source-visible, not open source. See `LICENSE` at the repository root.
