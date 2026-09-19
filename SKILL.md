---
name: "schelling-campaign-map"
description: "Turn a board game image into a Voronoi point-crawl campaign map, starting from the board's salient Schelling points. Survey → emit → seed → Voronoi → render, with a headless verification script for unsupervised harnesses."
metadata: { "includeInPrompt": true }
---

# Schelling Campaign Map

Turn a board game image into a Voronoi point-crawl campaign map, **starting from the board's salient Schelling points** — the locations every player would naturally coordinate on. No random seeds, no filler. The board tells you where the regions go. (With no board to read, Poisson-disc sampling with semantic anchoring stands in — see "Without a source image.")

## Purpose

Given a board game image (or map), produce a campaign map where every region is anchored to something the players already recognize. The method works for board games, city maps, and any image with salient locations.

When there is **no source image** — e.g. building a map from a prose setting — the survey step is replaced (see "Without a source image" below): seeds come from Poisson-disc sampling with semantic anchoring instead of measured markers.

## Workflow

### 0. Survey the board

Extract Schelling points in this order (see `references/method.md` for detail):

1. **Figures → capitals.** Character pieces on the board. Measure the figure's base.
2. **Named locations → supply centers.** Tiles carrying a location's name or symbol. The symbol-bearing tile IS the canonical marker.
3. **Special spaces → waypoint seeds.** Hazards, boons, "lose a turn," picture destinations. Free Schelling points.
4. **Distinct geography → wild/water seeds.** Seas, forests, mountains.

For each: crop tight, measure center, record `(fx, fy)` normalized 0–1 origin top-left. Verify with an overlay before committing. Sweep for misses — there is always one more.

For unsupervised harnesses, use the headless verify instead: `references/headless.md`, `bin/survey_auto.py`.

### 1. Emit the survey

Write points as structured data: name, kind, `(fx, fy)`, one-line provenance. Names come from the board — never invent toponyms.

### 2. Build the seeds

- **Special spaces go DIRECTLY on measured square centers.** Never snap to a hand-traced path (observed deviation up to 1.4 units drags seeds off their squares).
- Seed order: supply centers, then wilds/waters, then waypoints. Keep it stable.

### 3. Voronoi → render

- Tessellate (finite Voronoi, clipped to board). Fix the outward normal with the cloud-center sign rule: `sign(dot(ridge midpoint − cloud center, normal)) × normal`. Never use midpoint-minus-seed (it's perpendicular to the normal).
- Waypoints are **neutral**: never claimed, never quest hosts. They buffer borders.
- Render every marker **exactly at its seed**, never at cell centroids.

## Without a source image

When there is no board or map to survey — e.g. building a map from a prose setting — the survey step is replaced:

- **Poisson-disc sampling** (Bridson's algorithm) generates the seeds: an even, non-overlapping distribution with a minimum separation radius. Pick the radius so the region count fits comfortably (for ~14 regions in the unit square, r ≈ 0.2).
- **Semantic anchoring:** assign each sampled point to the named region whose authorial anchor position it lies nearest to — greedy nearest-anchor matching, in seed order (supply centers, then wilds/waters, then waypoints). The map keeps its intended geography (mines west, storm north) while the cells stay balanced and readable.
- Names still come from the source text — never invent toponyms.
- Record the method in `*_data.py` as the provenance: algorithm, radius, point count, and the anchor list. The even-distribution property replaces the measured-placement claim; the verify gate still checks names-against-source and in-bounds seeds.

## Output Contract

- `*_data.py`: survey output (names, kinds, coordinates, provenance).
- `*_pipeline.py`: Voronoi + render.
- `*_map.png`: the campaign map. `*_summary.json`: region list.
- Optional: quest matrix, faction claims — only if the campaign needs them.

## Operating Rules

1. Seeds are hand-placed at canonical markers when a source image exists. Random/filler seeding is rejected — except Poisson-disc sampling under "Without a source image" above.
2. Special-space seeds use measured centers with no path-snapping.
3. Waypoints are neutral and unclaimable.
4. Every marker sits on its seed. Every name was read off the board.
5. Run `bin/survey_auto.py verify` before every render in headless mode. Never render on a failed verify.

## Tooling

- `bin/survey.py` — crop/overlay helpers for the manual measure-and-verify loop.
- `bin/survey_auto.py` — headless detect + verify. `detect --board <img>` finds candidate squares by HSV color/shape; `verify --board <img>` checks every seed sits on the right color. Exits non-zero on failure. Tune `HSV_RANGES` per board.
