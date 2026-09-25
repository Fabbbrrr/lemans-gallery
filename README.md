# Day + night gallery (`docs/gallery`)

Phone-friendly gallery of **offline renders** of the whole track, twice per run: **DAY** (sun/sky at a good
hour) and **NIGHT** (the venue floodlit, exactly as `extension/ext_config.ini` drives it in game). A fast way
to see whether a build's lighting got better or worse — review 7's night was "whole track white", Fab's next
pass was "too dark, barely a difference under the posts".

* `index.html` — the gallery page (dark theme, 2 columns at phone width, tap a thumbnail for the full frame).
  Each run is two groups, DAY then NIGHT. Newest run first; every run stays on the page, so builds can be
  compared side by side. No JS, no CDN.
* `day/<date>-<commit>/` and `night/<date>-<commit>/` — `full/` JPEGs, 1280×720, quality ~82, ≤ 250 KB each;
  `thumbs/` 480 px wide for the grid; `meta.json` with the commit, timestamp, camera stations and the exact
  lighting values used.
* `.nojekyll` — stops GitHub Pages from running Jekyll over the folder.

Live: **https://fabbbrrr.github.io/lemans-gallery/**

## Render it (one command, ~4 min for both sets)

```bash
PATH=$HOME/bin311:$PATH python3 scripts/08n_night_gallery.py
```

32 images: each of the 16 camera stations (13 named sections of `track/lemans_lakeside/data/sections.ini`,
plus a section longer than ~100 m gets extra stations, plus three overviews — aerial, low wide from the
start/finish, and one from the pit lane) is rendered once per set.

Useful flags:

```bash
--samples 16            # default 16 (16-32 is plenty with denoising)
--spot-w 30000          # per-pole Cycles watts (default 30000) - the brightness knob
--set day|night|both    # default both
--run-id 2026-09-25-ab12cd3   # default <today>-<git rev-parse --short HEAD>
--only 04-,90-          # render a few shots only (calibration)
--publish               # render, then run scripts/publish_gallery.sh
NIGHT_SKY=1.15 ...      # night ambient-floor knob (default 1.15)
```

Re-running on the same date + commit **replaces that run's folders** (idempotent). A new commit or a new day
appends new folders and the index gains a new section — nothing older is deleted.

## Publish it

```bash
scripts/publish_gallery.sh              # gh must be authenticated as Fabbbrrr
```

Syncs the **contents** of `docs/gallery/` (index.html, `.nojekyll`, `day/**`, `night/**`) to the public repo
`Fabbbrrr/lemans-gallery` as its root, commits, pushes, and enables GitHub Pages on `main` / `/`, then polls
the URL until the index plus **three day and three night images** all return 200 (Pages can lag 1-2 min after
a push). The mirror clone lives in `dist/gallery_publish` (gitignored). Override with `GALLERY_REPO=` /
`GALLERY_PUB_DIR=`.

The gallery is kept **in this repo** because it regenerates with the track — the public repo only ever holds
renders.

## What the night scene actually is

The renders are a preview of the in-game CSP lighting, not an artistic impression. Everything comes from the
same sources the track ships:

* **The pole ring.** `config.json -> structures.floodlight_poles`: the 8 measured bases (shadow analysis) plus a
  **generated ring** — `spacing_m` 26 m, `offset_m` 9 m beyond the edge, alternating sides, jittered, pushed
  out where the run-off/tyre wall needs the room, skipped within `dedupe_m` of an existing pole — **27 poles**
  for the 581 m lap. `scripts/common.py: floodlight_positions()` builds that list once; `06_build_blender.py`
  (geometry), `07_package.py` (the `[LIGHT_n]` block) and this renderer all call it, so they cannot drift.
  Every clearance (racing surface, run-off, tyre wall, pit lane) is asserted, so a config edit that would drop
  a post on the racing surface fails the build.
* **The aim and the cone.** Each lamp aims at its nearest centreline point with the same maths as
  `[LIGHT_n]`. `csp_lights`: **115° cone**, **68 m range**, `fade_at_m` 260, `fade_smooth_m` 160,
  `diffuse_concentration` 0.5, `SPOT_SHARPNESS` 0.15, colour `3.0/2.85/2.6` — wide, long and soft, at a
  **modest 0.35** per pole, so ~27 overlapping cones sum to an even wash instead of 8 bright pools with dark
  gaps. `lamp_emissive` stays subtle at 3.0.
* **Lifting the areas between the poles.** The only documented lever CSP gives a track for this is
  `[LIGHTING] LIT_MULT` ("multiplier for dynamic lights affecting the track" — CSP wiki *Tracks – General
  options*, section `[LIGHTING]`; the same key is listed in the community config reference). It is shipped at
  **1.15**. There is **no documented track-side key that raises the venue's night ambient**:
  `TRACK_AMBIENT_GROUND_MULT` (same section) only redefines the ambient multiplier *for surfaces facing down*
  and would darken the track, so it is deliberately **not** used; `BOUNCED_LIGHT_MULT` only scales Extra FX
  bounce and needs Extra FX on. The brute-force route — more, wider, softer lights — is what the ring does.
* **The ambient floor is a render-only stand-in.** Cycles has no AC ambient, so the world background is the
  floor: `NIGHT_SKY` `(0.055, 0.075, 0.115)` at strength **1.15**, enough that no frame is pitch dark (the
  8-pole pass used 0.45 of a much darker blue). `meta.json` records both the sky and the CSP values.
* **The Cycles watts mapping.** CSP's `intensity` is its own unit, so the per-pole **Cycles watts (30000)** is
  the one number that is a mapping; it is chosen so the track reads as one floodlit surface with a faint
  sheen and nothing blows out to white. Recorded in `meta.json -> lighting.spot_power_w`.
* The day set uses the same sun/sky look as `08c_sections.py` (sun 42° elevation, azimuth 205°, sky
  `0.55/0.66/0.85` at 0.95) so the daylight frames match the other offline renders.
* Cycles CPU, denoised, `Standard` view transform.

## Constraints worth keeping

* Renders come from the **CC0 open build only** (`ground.mode = "open"`). The satellite/Esri drape is
  personal-use and must never appear in a published render; `meta.json` records the ground mode for each run.
* The renderer does not touch track geometry, materials or `config.json` — if a render exposes a defect,
  report it in `docs/REVIEW.md` rather than fixing it here.
