# Night gallery (`docs/gallery`)

Phone-friendly gallery of **offline night renders** of the whole track — a fast way to see whether a build's
night look got better or worse (after review 7 the floodlights had to stop flooding the place white).

* `index.html` — the gallery page (dark theme, 2 columns at phone width, tap a thumbnail for the full frame).
  Newest run first; every run stays on the page, so builds can be compared side by side.
* `night/<date>-<commit>/full/` — JPEGs, 1280×720, quality ~82, ≤ 250 KB each.
* `night/<date>-<commit>/thumbs/` — 480 px wide, for the grid.
* `night/<date>-<commit>/meta.json` — commit, timestamp, camera stations, and the exact lighting values used.
* `.nojekyll` — stops GitHub Pages from running Jekyll over the folder.

Live: **https://fabbbrrr.github.io/lemans-gallery/**

## Render it (one command, ~2.5 min)

```bash
PATH=$HOME/bin311:$PATH python3 scripts/08n_night_gallery.py
```

16 images: one chase-cam per named section of `track/lemans_lakeside/data/sections.ini` (13 today; a section
longer than ~100 m gets extra stations), plus three overviews — aerial at dusk, low wide from the
start/finish, and one from the pit lane.

Useful flags:

```bash
--samples 32            # default 24 (24-32 is plenty with denoising)
--spot-w 150000         # per-pole Cycles watts (default 120000) - the brightness knob
--run-id 2026-09-25-ab12cd3   # default <today>-<git rev-parse --short HEAD>
--only 04-,90-          # render a few shots only (calibration)
--publish               # render, then run scripts/publish_gallery.sh
NIGHT_SKY=0.6 ...       # night ambient knob (default 0.45)
```

Re-running on the same date + commit **replaces that run's folder** (idempotent). A new commit or a new day
appends a new folder and the index gains a new section — nothing older is deleted.

## Publish it

```bash
scripts/publish_gallery.sh              # gh must be authenticated as Fabbbrrr
```

Syncs the **contents** of `docs/gallery/` (index.html, `.nojekyll`, `night/**`) to the public repo
`Fabbbrrr/lemans-gallery` as its root, commits, pushes, and enables GitHub Pages on `main` / `/`, then polls
the URL until it returns 200 (Pages can lag 1-2 min after a push). The mirror clone lives in
`dist/gallery_publish` (gitignored). Override with `GALLERY_REPO=` / `GALLERY_PUB_DIR=`.

The gallery is kept **in this repo** because it regenerates with the track — the public repo only ever holds
renders.

## What the night scene actually is

The renders are a preview of the in-game CSP lighting, not an artistic impression:

* 8 spot lights at the `config.json` `structures.floodlight_poles` positions, aimed at each pole's **nearest
  centreline point** with the same maths as the `[LIGHT_n]` block in `scripts/07_package.py`, so the render
  and `extension/ext_config.ini` agree.
* `csp_lights` values are honoured where Cycles has an equivalent: colour `3.0/2.85/2.6`, **65° cone**
  (`spot_size`), **42 m range** (`cutoff_distance`), soft edge (`spot_blend` 0.35 ≈ `SPOT_SHARPNESS 0.4`),
  colour/cone/geometry straight from config.
* CSP's `intensity` is its own unit, so the per-pole **Cycles watts** is the one number that is a mapping
  (120000 W today) — picked so the track reads as pools of light with dark gaps and nothing blows out to
  white (the review-7 failure mode). It is recorded in `meta.json` under `lighting.spot_power_w`.
* The 13 corner shots use the night sky; the 3 overviews are lifted to blue hour so the whole site is legible.
* Cycles CPU, denoised, `Standard` view transform.

## Constraints worth keeping

* Renders come from the **CC0 open build only** (`ground.mode = "open"`). The satellite/Esri drape is
  personal-use and must never appear in a published render; `meta.json` records the ground mode for each run.
* The renderer does not touch track geometry, materials or `config.json` — if a render exposes a defect,
  report it in `docs/REVIEW.md` rather than fixing it here.
