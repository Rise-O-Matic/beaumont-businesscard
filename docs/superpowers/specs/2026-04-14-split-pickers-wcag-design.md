# Split Front/Back Pickers + WCAG Contrast Enforcement

**Date:** 2026-04-14
**Status:** Approved

## Problem

The current picker system has a single flat row of 6 swatches that conflate front and back face controls. The "Logo" swatch changes behavior depending on whether the front background is dark (logo forced to White) or light, but this isn't visible to the user. The back logo color is silently derived from `accent1` with no explicit control. Additionally, nothing prevents the user from selecting low-contrast combinations like Desert Sand text on a white background.

## Design

### New State Model

The state expands from 6 keys to 9:

```js
{
  // Front face
  frontBg:      'White / Cream',   // background gradient
  frontLogo:    'San Gorgonio Blue', // logo PNG color (forced White on dark BGs)
  frontAccent1: 'San Gorgonio Blue', // accent strip left + divider rule left
  frontAccent2: 'Sage',              // accent strip right + tagline text

  // Back face
  backLogo:     'San Gorgonio Blue', // back logo PNG (independent)
  backAccent1:  'San Gorgonio Blue', // bar left + job title + icons + divider border
  backAccent2:  'Sage',              // bar right + contact labels
  backText:     'San Gorgonio Blue', // name + contact text + link color
  backFooter:   'San Gorgonio Blue', // footer wave fill
}
```

Back background remains hardcoded: `#FDFAF5` for dark front themes, `#FDFCFA` for light.

### Picker Layout

Two labeled rows replacing the single row:

**Row 1 — "FRONT"** label, then 4 pickers:
| Front BG | Logo | Accent 1 | Accent 2 |

**Row 2 — "BACK"** label, then 5 pickers:
| Logo | Accent 1 | Accent 2 | Text | Footer |

Each row has a subtle uppercase label on the left (same `picker-label` font styling). Rows stack vertically with a small gap. On narrow screens they wrap naturally.

When `frontBg` is a dark option, the front Logo swatch shows white and the dropdown displays "White (auto)" as the locked selection since the logo is forced to White.png.

### WCAG Contrast Enforcement

#### Contrast Calculation

A `contrastRatio(hex1, hex2)` function using WCAG 2.1 relative luminance formula. Returns a ratio like 4.5, 7.0, etc.

#### Reference Background Colors

- **Front:** worst-case sample from the `frontBg` gradient. Light options check against `#FFFFFF`. Dark options check against their darkest gradient stop.
- **Back:** `#FDFCFA` (light front) or `#FDFAF5` (dark front) — both near-white.

#### Element Categorization

**Hard-filtered** (grayed out and unselectable in dropdown when they fail threshold):

| Picker | Threshold | Rationale |
|--------|-----------|-----------|
| Front Logo | 3:1 vs front BG | Large graphic element |
| Front Accent 2 (tagline) | 4.5:1 vs front BG | Small text |
| Back Logo | 3:1 vs back BG | Small graphic element |
| Back Accent 1 (job title + icons) | 3:1 vs back BG | Large text + UI elements |
| Back Accent 2 (labels) | 4.5:1 vs back BG | Small text |
| Back Text | 4.5:1 vs back BG | Small text |

**Warning only** (orange border on swatch, still selectable):

| Picker | Rationale |
|--------|-----------|
| Front Accent 1 | Decorative strip gradient |
| Back Footer | Decorative wave |

#### Behavior When Background Changes

If the user changes the front background and an already-selected color now fails its contrast threshold, the swatch shows a warning indicator (orange border). The user must manually pick a new compliant value.

#### Randomize Behavior

All randomly selected colors must pass their respective WCAG contrast thresholds. No warnings, only fully compliant combinations generated.

### Persistence & Sharing

**localStorage:**
- Key remains `bld-card-versions`.
- State objects use the new 10-key format.
- On first load of this version, any existing saved schemes are cleared (pre-release, no migration needed).

**URL hash:**
- 9-value JSON array: `[frontBg, frontLogo, frontAccent1, frontAccent2, backLogo, backAccent1, backAccent2, backText, backFooter]`
- Old 6-value URLs silently fail to load (length check won't match 9).

**Reset button:** Sets the new default state with split naming.

**Copy Full Spec:** Updated to show front and back sections with independent color assignments.

### What Does NOT Change

- Card dimensions, aspect ratio, scale behavior
- Font choices (DM Serif Display, News Cycle, Libre Franklin)
- Logo file paths and `LOGO_MAP` / `logoBase`
- `BG_OPTIONS` and `BRAND` palette objects
- Card HTML templates (`cardFrontHTML`, `cardBackHTML`)
- Saved card grid, 3D flip behavior, max 20 saves
- Print/PDF export styles
- Dark background behavior (white logo override, different back BG, wave alpha values)
