# mdplots

A local LaTeX style library for producing publication-quality figures from molecular dynamics simulation data. Uses **pgfplots** for rendering. Data preparation (downsampling, unit conversion, column naming) is handled by a separate Python pipeline.

---

## Quick start

### 1. Make the library findable

Add the `mdplots/` root directory to `TEXINPUTS` in your project's `.latexmkrc`:

```perl
$ENV{TEXINPUTS} = '/path/to/mdplots/:' . ($ENV{TEXINPUTS} // '');
```

Then in your document:

```latex
\usepackage{mdplots}
```

Alternatively, for a one-off test, compile from inside `examples/` — the `.latexmkrc` there sets `TEXINPUTS` automatically:

```bash
cd examples/
latexmk -pdf minimal.tex
```

### 2. Use a template

```latex
\mdplotMSD{
  \addplot table[x=t, y=msd] {data/msd_run01.dat};
  \addplot table[x=t, y=msd] {data/msd_run02.dat};
}{legend pos=north west}
```

---

## Template reference

All templates share the same two-argument signature:

```
\mdplot<Type>{plot-commands}{extra-axis-options}
```

`plot-commands` is everything that would go between `\begin{axis}` and `\end{axis}` — typically one or more `\addplot` statements plus `\addlegendentry` calls. `extra-axis-options` is appended to the pgfplots axis options and can override anything.

| Macro | Axes | Default labels | Typical use |
|---|---|---|---|
| `\mdplotMSD` | log-log | $t$ (ps) vs MSD (Å²) | Mean square displacement |
| `\mdplotRDF` | linear | $r$ (Å) vs $g(r)$ | Radial distribution function |
| `\mdplotEteACF` | semilog-x | $t$ (ps) vs $C_\mathrm{ee}(t)$ | End-to-end vector ACF |
| `\mdplotISF` | semilog-x | $t$ (ps) vs $F(q,t)$ | Intermediate scattering function |
| `\mdplotStressACF` | linear | $t$ (ps) vs $C_\sigma(t)$ | Stress autocorrelation |
| `\mdplotGeneric` | linear | _(none)_ | Any other 2D plot |

---

## Data contract

The Python export pipeline must produce files with these conventions:

- Whitespace-separated with a single named header line (e.g. `t msd msd_err`)
- Time always in **ps**; length always in **Å**
- Pre-downsampled to **≤ 2000 points** per series — use logarithmic binning for log-scale plots

---

## Width presets

Call one of these before the plot macro to switch figure widths:

```latex
\mdplotSetPaperSingle   % 8.5 cm  — default
\mdplotSetPaperDouble   % 17.0 cm
\mdplotSetThesis        % 14.0 cm
\mdplotSetSlides        % 11.0 cm
```

Or set `\mdplotWidth` directly: `\setlength{\mdplotWidth}{9cm}`.

---

## Figure caching (external library)

For documents with many figures, enable pgfplots' external library to cache compiled figures as PDFs. This requires `-shell-escape` and a `figures/cache/` directory.

```latex
% In the preamble, after \usepackage{mdplots}:
\mdplotEnableExternal
```

```bash
mkdir -p figures/cache
pdflatex -shell-escape main.tex
```

---

## Colors

The default cycle list uses the **Okabe-Ito** colorblind-safe palette. Named colors `mdOI1`–`mdOI8` are available for direct use (e.g. in fill-between error bands).
