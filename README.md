# IR Explorer

Browser-based tool for auditioning and blending guitar impulse responses. Record a DI'd rhythm part, record a lead over it, blend up to 8 IRs per side interactively while the loop plays, and export the blended result as a `.wav` IR.

**Live:** https://mwmfhk.github.io/IRExplorer/ir-explorer.html

## Usage

1. Press **Record** on the Rhythm panel and capture a DI'd rhythm loop. Press **Stop**, then **Play** to loop it.
2. Press **Record** on the Lead panel and play a lead over the rhythm. Playback re-syncs both loops automatically.
3. Click **Add IR** to load `.wav` IR files. Each IR sits at a polygon vertex; drag the orange point inside the polygon to interpolate between them.
4. Adjust per-side Gain / EQ / Reverb / phase-correction as needed.
5. Click **Export IR** to save the currently-blended IR as a `.wav`.
6. **Save session** bundles both recordings, all loaded IRs, and every knob position into a single `.irz` file (a zip). **Load session** restores it. **New session** clears everything.

Works on desktop Chrome / Safari / Edge and on iOS Safari (iPhone + iPad).

## Repo

- `ir-explorer.html` — the app; single-file, no build step, open directly in a browser.
- `algorithms.html` — technical write-up of the DSP, sync, and file formats. Live at [algorithms.html](https://mwmfhk.github.io/IRExplorer/algorithms.html).
- `ir-session-pr100.irz` — demo session that loads on first open.

## License

Apache 2.0 — see `LICENSE`.
