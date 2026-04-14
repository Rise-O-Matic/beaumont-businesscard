# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file business card color builder for the **Beaumont Library District (BLD)**. Users customize card colors via dropdown pickers and save schemes to localStorage. Saved schemes appear below the builder as 3D flip cards in a responsive grid.

## Architecture

Everything lives in **index.html** — no build step, no dependencies, no framework. Open the file directly in a browser.

### Structure within index.html

1. **CSS** (~620 lines): Card system (700×400 at 1.75:1 ratio), builder UI (pickers, buttons), saved section (CSS 3D flip cards in auto-fill grid), responsive breakpoints at 1480px/740px/380px, print styles
2. **HTML**: Builder preview (front + back card with `id="custom-*"` elements for JS targeting), pickers container, action buttons, saved-section grid
3. **JavaScript** (~500 lines): Brand palette (`BRAND` object), background options (`BG_OPTIONS`), state management (6-property `state` object), `applyState()` for builder preview, `applyStateToCard()` for saved cards, localStorage persistence, URL hash sharing

### Key Patterns

- **Builder preview** uses `id`-based selectors (e.g., `#custom-front`, `#custom-strip`) — there's only one preview card
- **Saved cards** use class-based selectors scoped to each card element (e.g., `cardEl.querySelector('.saved-flip-front .accent-strip')`) — multiple cards coexist
- Saved cards render at full 700×400 size then `transform: scale(0.5)` inside 350×200 containers
- Logo PNGs live in `BLD Logo Suite (2)/BLD Logo Suite/Horizontal/PNG Transparent Background/` — the path has spaces and parentheses, always use the `logoBase` variable

### Brand Palette

The canonical palette is the `BRAND` object in JS. Key colors: San Gorgonio Blue (#15424A), Sage (#5A7A6A), Rust (#8B5E3C), Lavender Purple (#7A5480). The default state matches the Original BLD Website Theme (mybld.org).

### State Model

```
{ frontBg, logo, accent1, accent2, textColor, footer }
```

`accent1` drives: accent strip primary, icons, job title, divider. `accent2` drives: tagline, labels, name label. Not every original BLD color mapping is reproducible through this 2-accent model.
