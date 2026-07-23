# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A single-file business card color builder for the **Beaumont Library District (BLD)**. Users customize card colors via dropdown pickers and save schemes to localStorage. Saved schemes appear below the builder as 3D flip cards in a responsive grid.

## Development

No build step, no dependencies, no framework. Open `index.html` directly in a browser. Refresh to see changes. No tests, no linter.

For a live preview (and the only reliable way to load the page in headless/automation tooling, since `file://` doesn't render correctly there), run `preview.cmd` (Windows, double-click) or `./preview.sh` (Git Bash/macOS/Linux). Both serve the card at `http://localhost:8731/index.html` and open it in your browser; press Ctrl+C to stop.

External CDN libraries: `html2canvas` (card-to-image capture), `jsPDF` (print PDF export with crop marks and bleed), and `qrcode-generator` (back-face QR code, encoded once at load into a data URI).

## Architecture

Everything lives in **index.html** (~2200 lines total):

1. **CSS** (~800 lines): Card system (700x400 at 1.75:1 ratio), builder UI (pickers, buttons, export dropdown), saved section (CSS 3D flip cards in auto-fill grid), responsive breakpoints at 1480px/740px/380px, print styles
2. **HTML** (~92 lines): Builder preview (front + back card with `id="custom-*"` elements), picker containers under each card, action buttons, saved-section grid with flip-all/import/export controls
3. **JavaScript** (~1250 lines): Brand palette, background options, WCAG contrast checking, state management, picker UI construction, localStorage persistence, URL hash sharing, scheme name generator, PDF export with crop marks

### Key Patterns

- **Builder preview** uses `id`-based selectors (e.g., `#custom-front`, `#custom-strip`) -- there's only one preview card
- **Saved cards** use class-based selectors scoped to each card element (e.g., `cardEl.querySelector('.saved-flip-front .accent-strip')`) -- multiple cards coexist
- These are two parallel rendering paths: `applyState()` updates the builder preview via `getElementById`, `applyStateToCard(cardEl, s)` updates a saved card via `querySelector` within that card's DOM subtree
- Saved cards render at full 700x400 size then `transform: scale(0.5)` inside 350x200 containers
- Logo PNGs live in `BLD Logo Suite (2)/BLD Logo Suite/Horizontal/PNG Transparent Background/` -- the path has spaces and parentheses, always use the `logoBase` variable
- `LOGO_MAP` translates brand color names to logo filenames (they don't always match, e.g., `'Lavender Purple'` maps to `'Lavender Fields Purple'`)
- The Name, Job Title, Phone, and Email fields on the back face are `contenteditable` and each stays on one line. `autoScaleFit(el, MAX, MIN)` shrinks a line until it fits its column: `autoScaleName()` runs the name from 2.4rem down to 1.1rem across the full 610px identity width, `autoScaleDetails(scope)` runs the job title / phone / email from 1.6rem down to 1.05rem inside the 470px column beside the QR
- `autoScaleFit()` no-ops (and clears the inline size) when its `.card-face` isn't 700px wide — below the 740px breakpoint the card is sized in vw and the media query owns the type scale

### WCAG Contrast System

Each picker has a `CONTRAST_RULES` entry defining a minimum contrast ratio, an enforcement mode, and the background it's checked against (`bgSource: 'front' | 'back' | 'band'`):
- `mode: 'hard'` -- the randomizer will only pick colors that pass (logos, text, back-face labels)
- `mode: 'warn'` -- swatch gets a `.contrast-warn` CSS ring, but the color remains selectable (accent strip, rule/wave)

`BG_CHECK_COLORS` maps each background option to its worst-case hex for contrast math. `getCheckBgs()` returns *every* background a color must clear -- for `bgSource: 'band'` (the back logo) that's all four stops of the band gradient. `passesContrast()` and `contrastRatio()` implement WCAG relative luminance checking.

When `frontBg`, `backAccent1`, or `backAccent2` changes, all picker dropdowns are rebuilt via `refreshSwatchUI()` because contrast thresholds shift (the back accents define the band the logo sits on). Individual picker changes only call `updateContrastWarnings()`.

### Dark Background Behavior

When `BG_OPTIONS[frontBg].dark` is true: front logo forces `White.png` (picker locked/disabled) and wave SVGs use different alpha values. The back face always uses a light background (`#FDFAF5` for dark themes, `#FDFCFA` for light).

The back band has the same behavior driven by its own colors: `bandIsDark(bc1, bc2)` is true when white clears 3:1 against every stop of `bandStops()`, and then the back logo forces `White.png` (picker locked to "White (auto)") and the branch address reverses to white. Otherwise the chosen `backLogo` color is used and the address falls back to `backText`. The band gradient is built by `bandGradient()` with `BAND_LIGHTEN = 18` -- a much shallower sweep than the front accent strip's `lighten(...,40)`, because the whole band has to stay legible under the logo and address.

### State Model

```js
{
  frontBg,                     // Background option key (from BG_OPTIONS)
  frontLogo, frontAccent1, frontAccent2,  // Front face: logo + 2 accents (from BRAND/LOGO_COLORS)
  backLogo, backAccent1, backAccent2,     // Back face: logo + 2 accents
  backText, backFooter                    // Back face: text color + footer wave color
}
```

9 keys total. Front and back have independent pickers. `frontAccent1` drives: accent strip gradient primary, bottom wave tint. `frontAccent2` drives: accent strip gradient second hue, front channels (web/social) text. `backAccent1` drives: back band gradient primary, job title. `backAccent2` drives: back band gradient second hue. `backText` drives: name, phone/email lines, and the branch address when the band is light. `backFooter` (labelled "Rule & Wave") drives both the thin rule under the band and the footer wave fill.

The back face is a letterhead layout, restored from the customer's original card:

- `.back-bar` — a 156px deep gradient band across the top (bleeds via `::before`), holding the logo (`.back-logo-small`, 250px wide at top 16 / left 30) and the branch address (`.back-org`, News Cycle 700 at 1.6rem, right-anchored, vertically centred in the band)
- `.back-rule` — a 10px accent rule under the band, bleeding left and right
- `.back-identity` — name at 2.4rem across the full 610px width, then `.back-details` (job title in Libre Franklin 600, phone and email in News Cycle 700, all 1.6rem) capped at 470px so the stack clears the QR
- `.back-qr` — 116px QR block bottom-right, bottom-aligned with the email line
- `.back-footer` — the wave, unchanged

Everything on the back is positioned in px against the 700px card; the `max-width: 740px` media query restates those values in vw (1 card px = 0.1314vw at a 92vw card).

### Persistence

- **localStorage** key: `bld-card-versions`. Array of `{ name, date, state }` objects. Max 20 saved schemes. A migration on load clears pre-v2 saves (old 6-key state format without `frontLogo`).
- **URL hash sharing**: `#custom=` + URI-encoded JSON array of the 9 state values in order. Loaded on page init.
- **Collection export/import**: JSON file download/upload of the full saved array.

### PDF Export

Uses `html2canvas` to capture front and back at 3x scale with 0.125" bleed, then `jsPDF` to compose pages with crop marks, registration marks, and a slug line. Output is a 2-page landscape PDF at the print-ready trim size (3.5" x 2").

### QR Code

`qrcode-generator` encodes `QR_TARGET` (`https://www.myBLD.org`) once at load into `qrDataURL` (a GIF data URI at 16px modules, so it downscales cleanly into the 116px block and survives the 3x html2canvas capture). The builder card gets it via `paintQR()`; saved cards and the actual-size preview get it inline from `cardBackHTML()`. If the CDN is unavailable `qrDataURL` is empty and the QR block is omitted rather than rendering broken.

### Brand Palette

The canonical palette is the `BRAND` object in JS. Key colors: San Gorgonio Blue (#15424A), Sage (#5A7A6A), Rust (#8B5E3C), Lavender Purple (#7A5480). The default state matches the Original BLD Website Theme (mybld.org).

`LOGO_COLORS` is a filtered subset of `BRAND` -- only colors that have matching logo PNG files in `LOGO_MAP`.

Defaults: the front runs San Gorgonio Blue + Sage; the back sets *both* accents to San Gorgonio Blue so the band reads as one solid color, with Sage on `backFooter` so the rule and footer wave stand off it.

### Fonts

Three Google Fonts loaded via CDN: `DM Serif Display` (headings/names), `News Cycle` (body text), `Libre Franklin` (labels/taglines/subtitles).

### Scheme Name Generator

`generateSchemeName()` builds evocative names from `COLOR_WORDS` (per-brand-color word lists), `BG_WORDS` (per-background word lists), and geographic `SUFFIXES`. Used as the default suggestion in the save prompt and as the PDF filename.
