# CLAUDE.md — R / Shiny Learning Project

## What this project is

Learning R and Shiny by rebuilding and extending analysis from Jake's BSc Biochemistry
dissertation (University of Salford, 2024). The dissertation compared atomistic (CHARMM36)
and coarse-grained (MARTINI 2.2 and 3.0) molecular dynamics simulations of pure DPPS
lipid bilayer membranes, using GROMACS. This project uses that real data and those real
findings as the substrate for learning R — not a toy dataset, not a tutorial exercise.

Not doing CS50R's final project. Learning by shipping small, complete, meaningful tools.

---

## The data

GROMACS output files from the dissertation simulations:

- **XVG files** — time-series data for area per lipid (APL), exported from `.edr` binary
  using `gmx energy`. Tab-separated with `#` and `@` comment headers that need skipping.
- **Density XVG files** — output from `gmx density`, one curve per component
  (membrane, water, Na⁺, headgroup atoms/beads).
- **Summary table** — six conditions × three metrics (APL nm², compressibility κA, thickness nm):

| Condition | APL (nm²) | Compressibility (κA) | Thickness (nm) |
|-----------|-----------|----------------------|----------------|
| C36       | 0.462191  | 1054.78              | 4.67864        |
| C36-sa    | 0.490671  | 565.5694             | 9.7504         |
| M3        | 0.592023  | 152.2413             | 4.6777         |
| M3-sa     | 0.592737  | 148.5925             | 4.66148        |
| M2        | 0.603063  | 192.4494             | 4.88812        |
| M2-sa     | 0.603984  | 172.7012             | 5.15066        |

Forcefields: C36 = CHARMM36 (atomistic), M3 = MARTINI 3.0 (CG), M2 = MARTINI 2.2 (CG).
Assembly: default = pre-assembled via CHARMM-GUI, `-sa` suffix = self-assembled.

---

## Mental model: Shiny is Flask

Use this framing throughout — Jake knows Flask well.

| Flask                              | Shiny                                      |
|------------------------------------|--------------------------------------------|
| `app.py` routes                    | `server.R` (or `server` function)          |
| Jinja2 templates                   | `ui.R` (or `ui` object)                   |
| Query params / form POST           | `input$id` reactive values                 |
| `render_template(data=...)`        | `renderPlot({})` / `renderTable({})`       |
| Manual state management            | Reactive dependency graph (automatic)      |
| `url_for('route')`                 | `outputId` linking ui ↔ server             |
| `if request.method == 'POST'`      | `observeEvent(input$button, {})`           |
| Blueprints                         | Shiny modules (later — don't need yet)     |

Key difference: Flask is request/response. Shiny is reactive — when an input changes,
everything that depends on it re-executes automatically. You don't wire that manually.

Single-file Shiny apps (`app.R` with `ui` and `server` in one file) are the equivalent of
a simple Flask app before you reach for blueprints. Start there.

---

## Project roadmap

### Project 1 — APL Visualiser *(start here)*
**Goal:** Load XVG files, recreate APL-over-time plots in ggplot2, wrap in Shiny.
**UI:** Dropdown to select forcefield (C36 / M3 / M2), toggle for pre- vs self-assembled.
**Key R to learn:** `readr::read_delim` with comment skipping, `dplyr` pipes,
`ggplot2` (geom_line, labs, theme_minimal), `shiny::selectInput` → `renderPlot`.
**Done when:** Shiny app runs locally, plots match dissertation figures by eye.

### Project 2 — Compressibility & Thickness Dashboard
**Goal:** Interactive comparison of all six conditions across all three metrics.
**UI:** Bar charts, radar/spider chart, sortable data table. Layout via `bslib`.
**Key R to learn:** `plotly`, `DT::datatable`, `tidyr::pivot_longer`,
`bslib::page_sidebar` layout.
**Done when:** Dashboard loads, all charts update on interaction, table is sortable.

### Project 3 — Self-Assembly Kinetics Explorer
**Goal:** Annotated Shiny app explaining the self-assembly timeline findings from
the dissertation. Slider to step through stages, APL convergence plotted alongside,
text annotations explaining what's happening at each stage.
**Key R to learn:** `sliderInput`, `ggplot2::geom_vline` + `annotate`,
`shiny::conditionalPanel`, possibly `Quarto` for embedding.
**Done when:** Someone unfamiliar with the dissertation can follow the story.

### Project 4 — Michaelis-Menten Curve Fitter *(parallel, standalone)*
**Goal:** Input Vmax and Km, fit and plot MM curve. Compare multiple enzyme variants.
**Standalone** — no dissertation data required. Keeps biochem thread alive.
**Key R to learn:** `nls()` for non-linear least squares fitting, `ggplot2::stat_function`,
reactive parameter sliders updating live plot.
**Done when:** Sliders move, curve updates live, fit stats displayed.

---

## R conventions for this project

```r
# Package loading — use pacman or explicit library() calls
library(tidyverse)   # dplyr, ggplot2, tidyr, readr, purrr
library(shiny)
library(bslib)       # modern Bootstrap theming (replaces shinydashboard)
library(plotly)      # interactive charts
library(DT)          # interactive tables
```

### Reading XVG files
XVG files have `#` comment lines and `@` directive lines before the data.
```r
read_xvg <- function(path) {
  read_delim(
    path,
    delim = " ",
    comment = "#",
    col_names = FALSE,
    trim_ws = TRUE
  ) |>
  filter(!str_starts(X1, "@")) |>
  mutate(across(everything(), as.numeric))
}
```

### Pipe style
Use the native R pipe `|>` (R 4.1+), not `%>%`. They're equivalent here, but `|>` has
no dependency.

### File structure (single-file app for Projects 1 and 4)
```
project-name/
├── app.R           # ui + server in one file
├── data/           # XVG files or CSV exports
├── R/              # helper functions (read_xvg, etc.)
└── CLAUDE.md       # this file, copied here
```

Multi-file (Projects 2–3 when complexity warrants):
```
project-name/
├── ui.R
├── server.R
├── global.R        # shared data loading, sourced by both
├── data/
└── R/
```

---

## Working methodology

- **AI-assisted development.** Jake uses Claude Code as the agentic build partner.
  Architectural decisions are Jake's — Claude Code executes them.
- **Read edits and shell commands** as they run. Don't just press enter and watch.
- **Conventional commits.** `feat:`, `fix:`, `refactor:`, `chore:`. One thing per commit.
- **Feature branches.** `feat/apl-visualiser`, `fix/xvg-parser`, etc.
- **Ship, then improve.** A working ugly app beats a perfect idea.

---

## What to name things explicitly

When suggesting R code or architecture, always name the idiom or pattern:
- "this is a `pivot_longer` — the tidyverse equivalent of reshaping from wide to long"
- "this is `nls()` — non-linear least squares, R's built-in curve fitter"
- "this `observeEvent` is the equivalent of a Flask route with `if request.method == 'POST'`"

Jake learns faster when the why and the what-it-maps-to are stated alongside the code.

---

## Background context

- BSc Biochemistry, University of Salford (2024). Dissertation: DPPS lipid bilayer MD
  simulation, atomistic vs coarse-grained. Tools: CHARMM-GUI, GROMACS, VMD, HPC.
- 10 years UK hospitality, de facto ops management of a 2-venue business.
- CS50x (Week 0–6), CS50P (Week 0–5) complete. Shipped: Flappy Bird PWA, TiltJump,
  Cocktail Shaker PWA, Flask bar website, cocktail delivery web app.
- Stack: Python, Flask, SQLite/PostgreSQL, Vanilla JS, HTML5 Canvas, Cloudflare.
- Syntax recall without docs is weak. System design and architectural thinking are strong.
  Treat as senior-developer thinking, junior-developer syntax confidence.
- Currently: digital nomad, Da Nang, Vietnam. Low burn rate. First online income is the
  90-day target.
