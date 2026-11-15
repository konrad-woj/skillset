---
name: scientific-figure-making
description: >-
  Publication-ready matplotlib figures for academic papers, slides, and
  reports: grouped bars, trends, scatter, heatmaps, radar, and multi-panel
  layouts, with a consistent house style (fonts, palette, spines, legends) and
  print/vector export (PDF/SVG/high-DPI). Wraps the upstream skill from
  github.com/ChenLiu-1996/figures4papers and fetches it on first use. Triggers
  on "figure for my paper", "publication-quality plot", "make this plot
  camera-ready", "matplotlib figure for slides", "export figure as PDF/SVG".
  Do not use for interactive or web viz (Plotly, Altair, Bokeh), EDA-only plots
  with no publication target, 3D/GIS-first work, or Illustrator/Figma layouts.
---

# Scientific figure making (external skill)

This skill is a pointer. The real content lives upstream in
[ChenLiu-1996/figures4papers](https://github.com/ChenLiu-1996/figures4papers)
under `scientific-figure-making/`, next to the `figure_*` demo scripts it links
to. Fetch it, then follow the upstream `SKILL.md` as if it were this file.

## External dependency

| Dependency | Source | Pinned ref |
| ---------- | ------ | ---------- |
| `figures4papers` | `https://github.com/ChenLiu-1996/figures4papers.git` | `0ddfbd3fc9892e804a3fa01ae46bd0d32617f2aa` |

The full repo is ~80 MB of images. Fetch only the skill folder, the demo
`.py` scripts, and the license (under 1 MB) with a sparse, blobless clone.

## Step 1: fetch (once per machine)

Use a cache directory outside the user's project so nothing gets committed:

```bash
F4P="${XDG_CACHE_HOME:-$HOME/.cache}/skillset/figures4papers"
F4P_REF=0ddfbd3fc9892e804a3fa01ae46bd0d32617f2aa

if [ ! -f "$F4P/scientific-figure-making/SKILL.md" ]; then
  rm -rf "$F4P"
  git clone --filter=blob:none --no-checkout \
    https://github.com/ChenLiu-1996/figures4papers.git "$F4P"
  git -C "$F4P" sparse-checkout set --no-cone \
    '/scientific-figure-making/' '/figure_*/*.py' '/LICENSE'
  git -C "$F4P" checkout --quiet "$F4P_REF"
fi
```

To pick up upstream changes, the user can bump `F4P_REF` in this file, or
delete the cache directory and rerun. Don't follow `main` without being asked.

If the clone fails (no network, host blocked), say so and continue with
general matplotlib best practice. Don't invent the upstream palette or helper
APIs.

## Step 2: use it

1. Read `$F4P/scientific-figure-making/SKILL.md` and do what it says.
2. Open files under `$F4P/scientific-figure-making/references/` only when
   that SKILL.md's table says to. Don't preload them all.
3. When `references/demos.md` points at a `figure_*` project, read the local
   script under `$F4P/figure_<name>/` instead of browsing GitHub.
4. Write the user's figure code in their project, not in the cache directory.

## License

figures4papers is licensed **CC BY-NC 4.0** (attribution, non-commercial).
Patterns and style ideas are fine to apply. If you copy substantial code from
the `figure_*` scripts or the reference docs into the user's project, add an
attribution comment linking the upstream repo, and flag the non-commercial
term if the project looks commercial.
