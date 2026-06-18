---
name: plotting
description: >-
  Personal conventions for creating or editing matplotlib/seaborn figures
  (Arial font, Nature/NPG categorical palette, fully-opaque colors for
  PowerPoint/PDF export, spine removal, high-DPI saving). Use whenever writing
  or modifying any plotting/figure code or generating figures.
---

# Plotting conventions

Apply these whenever you create or modify a figure. The transparency rule is the
one that bites repeatedly — treat it as non-negotiable.

## Font

Always Arial. Set it globally once (e.g. in the package `__init__` or at the top
of a script) so every plot inherits it:

```python
import matplotlib as mpl
mpl.rcParams["font.family"] = "Arial"
```

`seaborn`'s `set_style`/`set_palette` do **not** reset the font, so a global set
sticks. Per-figure `plt.rcParams["font.family"] = "Arial"` is fine as explicit
reinforcement but not required once it's set globally.

## Color palette

Use the Nature journal (NPG) palette for categorical series. Define the palette
once as a module constant (e.g. `LIST_NPG_COLORS`) and apply it:

```python
import seaborn as sns
sns.set_palette(sns.color_palette(LIST_NPG_COLORS))
sns.set_style("white")
```

Keep semantic color maps (e.g. one color per category) as named dict constants
near the data they describe, so colors stay consistent across figures. When
choosing custom highlight colors, verify they stay distinct from each other and
remain distinguishable under color-vision deficiency (a CIEDE2000 check works
well).

## Transparency — emit FULLY OPAQUE colors (critical)

Figures often get dropped into PowerPoint and exported to PDF. Any SVG
transparency (`opacity` / `fill-opacity` < 1) renders as a **grey hue** in that
pipeline.

`mcolors.to_rgba(color, alpha=0.5)` does **NOT** fix this — baking alpha into an
RGBA tuple still writes `fill-opacity` into the SVG. The only reliable fix is to
emit no alpha at all: composite the tint onto the (white) background to get an
opaque RGB with the same on-white appearance.

```python
import matplotlib.colors as mcolors

def flatten_alpha(color, alpha, bg=(1.0, 1.0, 1.0)):
    """Composite color at alpha over bg -> opaque RGB (no transparency)."""
    return tuple(alpha * c + (1 - alpha) * b
                 for c, b in zip(mcolors.to_rgb(color), bg))

# patches/cells: opaque fill, opaque edge — no alpha anywhere
facecolor = flatten_alpha("#FFD700", 0.5)
ax.add_patch(Rectangle((x, y), w, h, facecolor=facecolor, edgecolor="#FFD700"))

# legends: opaque frame (framealpha defaults to 0.8, which also greys)
ax.legend(..., framealpha=1.0, facecolor="white", edgecolor="black")
```

**Rule:** if any artist would have `alpha < 1`, pre-blend it to opaque first.
Never pass `alpha=` to fills, and set `framealpha=1.0` on legends.

Alternatives if opaque colors are not enough:
- Save a high-DPI PNG for slides:
  `fig.savefig("plot.png", dpi=300, bbox_inches="tight", facecolor="white")`.
- If SVG is required, set `mpl.rcParams["svg.fonttype"] = "none"` and
  `mpl.rcParams["svg.hashsalt"] = "42"`, then in PowerPoint drag the SVG →
  right-click → Convert to Shape.

## Spines

Remove spines while keeping tick marks for a cleaner look:

```python
for side in ("top", "right", "left", "bottom"):
    ax.spines[side].set_visible(False)
```

## Saving

Save with `bbox_inches="tight"` and, for slide-bound figures,
`facecolor="white"`. For analysis figures meant to be both edited and embedded,
save `svg` (vector, editable), `png` (high-DPI raster fallback), and `pdf`. If
the project has a structured output helper (e.g. a `save_figure()` that lands
files under a dated/structured tree), use it rather than ad-hoc paths.
