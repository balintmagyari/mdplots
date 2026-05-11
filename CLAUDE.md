# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this project is

`mdplots` is a **LaTeX style library** (not a Python package, not a CTAN package) for producing publication-quality figures from molecular dynamics simulation data. It uses **pgfplots** for rendering. Data preparation (downsampling, unit conversion, column naming) happens in a separate Python pipeline — this library only handles plotting.

## Current implementation status

All core files are implemented and tested. Both example documents compile cleanly with `latexmk`.

```
mdplots/
├── mdplots.sty                  # entry point — loads all sub-files
├── styles/
│   ├── mdplots-colors.sty       # Okabe-Ito palette + cycle list
│   ├── mdplots-fonts.sty        # \mdplotWidth length + preset commands
│   └── mdplots-axes.sty         # mdplots-base pgfplotsset style
├── templates/
│   ├── plot-generic.tex         # \mdplotGeneric   — linear axes, no default labels
│   ├── plot-msd.tex             # \mdplotMSD        — log-log, t vs MSD
│   ├── plot-rdf.tex             # \mdplotRDF        — linear, r vs g(r)
│   ├── plot-stress.tex          # \mdplotStressACF  — linear, t vs C_sigma
│   ├── plot-acf.tex             # \mdplotEteACF     — semilog-x, t vs C_ee
│   └── plot-isf.tex             # \mdplotISF        — semilog-x, t vs F(q,t)
└── examples/
    ├── .latexmkrc               # sets TEXINPUTS=../ for compilation from this dir
    ├── minimal.tex              # self-contained test (inline coordinates, no data files)
    ├── thesis-figure.tex        # multi-curve demo using data/ files
    └── data/                    # synthetic but physically realistic example data
        ├── msd_300K.dat
        ├── msd_350K.dat
        ├── rdf_OO.dat
        ├── rdf_OH.dat
        ├── acf_ete.dat
        ├── isf_q1.dat           # q = 1.0 Å⁻¹
        ├── isf_q2.dat           # q = 2.0 Å⁻¹
        ├── isf_q3.dat           # q = 3.5 Å⁻¹
        └── stress_acf.dat
```

## How to compile the examples

From the `examples/` directory — the `.latexmkrc` there sets `TEXINPUTS=../` automatically:

```bash
cd examples/
latexmk -pdf minimal.tex
latexmk -pdf thesis-figure.tex
```

To use the library in another project, add its root to that project's `.latexmkrc`:

```perl
$ENV{TEXINPUTS} = '/path/to/mdplots/:' . ($ENV{TEXINPUTS} // '');
```

Then `\usepackage{mdplots}` works from any `.tex` file in that project.

Shell-escape (`-shell-escape`) is only needed if `\mdplotEnableExternal` is called for figure caching.

## Architecture: three-layer design

| Layer | File(s) | Role |
|---|---|---|
| **Config** | `styles/mdplots-colors.sty`, `styles/mdplots-fonts.sty` | Colors, widths — single source of truth |
| **Axis preset** | `styles/mdplots-axes.sty` | `mdplots-base` pgfplotsset style (visual defaults) |
| **Templates** | `templates/plot-*.tex` | One macro per plot type; full `tikzpicture` + `axis` |

`mdplots.sty` uses the `currfile` package to resolve the absolute path of its own directory, then `\input`s all sub-files using that path. This means the library works correctly regardless of where the calling document lives — no TEXINPUTS manipulation needed for sub-files, only for finding `mdplots.sty` itself.

## Template macro signature

**All templates take two arguments:**

```latex
\mdplot<Type>{plot-commands}{extra-axis-options}
```

- `plot-commands`: everything between `\begin{axis}` and `\end{axis}` — one or more `\addplot` lines, `\addlegendentry` calls, etc.
- `extra-axis-options`: appended to the pgfplots axis options; can override anything

Example with multiple curves and a legend:

```latex
\mdplotMSD{
  \addplot table[x=t, y=msd] {data/msd_300K.dat}; \addlegendentry{300 K}
  \addplot table[x=t, y=msd] {data/msd_350K.dat}; \addlegendentry{350 K}
}{legend pos=north west, xmin=0.01}
```

## Width preset commands

Call before a plot macro to switch figure width:

```latex
\mdplotSetPaperSingle   % 8.5 cm  (default)
\mdplotSetPaperDouble   % 17.0 cm
\mdplotSetThesis        % 14.0 cm
\mdplotSetSlides        % 11.0 cm
```

Or set directly: `\setlength{\mdplotWidth}{9cm}`.

## Critical pgfplots constraint — xmode/ymode

**Never put `xmode=log` or `ymode=log` inside a named `\pgfplotsset` style.**

pgfplots requires scale-mode keys to be applied before axis initialisation completes. Named styles defined with `\pgfplotsset` are resolved through the TikZ unknown-key pathway when used in `\begin{axis}[mystyle]`, which means they arrive after the axis has already started. pgfplots throws:

```
! Package pgfplots Error: Sorry, you can't change `/pgfplots/xmode' in this context.
```

**The fix used throughout this library:** every template that needs log or semi-log axes sets `xmode=log` / `ymode=log` as **direct** `\begin{axis}` options alongside `mdplots-base`:

```latex
\begin{axis}[
  mdplots-base,
  xmode=log, ymode=log,   % direct options — not inside mdplots-base
  ...
]
```

`mdplots-base` only contains visual settings (colors, tick style, legend, line width) which are not subject to this constraint.

## Inline data in example files

`\addplot table { ... }` with multi-line inline data **does not work inside a macro argument** because LaTeX tokenises the argument and end-of-line characters become spaces. pgfplots then cannot parse row boundaries.

Use `\addplot coordinates {(x,y) ...}` for inline example data in `.tex` files. For real simulation data from `.dat` files this issue does not arise.

## Data contract (Python → LaTeX)

Files must follow these conventions for `\addplot table[x=col, y=col]` to work:

- Whitespace-separated, single named header line (e.g. `t msd`)
- Comment lines starting with `#` are ignored by pgfplots
- Time in **ps**, length in **Å** — never mix units
- Pre-downsampled to **≤ 2000 points** per series (logarithmic binning for log-scale plots)

## Colors

The default cycle list uses the **Tol Vibrant** colorblind-safe palette (7 colors). Named colors `mdC1`–`mdC7` are available for fills, e.g. in `fill between` error bands (the `fillbetween` pgfplots library is loaded automatically). Names are palette-agnostic so a future palette swap requires no renames elsewhere.

Cycle order: orange, blue, cyan, magenta, teal, red, grey. Each slot also pairs with a distinct mark (filled circle, square, triangle, diamond, pentagon, then `x`, `+`). The first two slots (orange + blue) are the canonical colorblind-safe pair for two-curve comparisons.

## Figure caching

To cache compiled figures as PDFs (large documents only):

```latex
% In preamble after \usepackage{mdplots}:
\mdplotEnableExternal
```

```bash
mkdir -p figures/cache
pdflatex -shell-escape main.tex
```
