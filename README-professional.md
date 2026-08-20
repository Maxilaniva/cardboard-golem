# Cardboard Golem

A dependency-free, single-file web application that renders Magic: The Gathering deck lists into dimensionally accurate, print-ready proxy sheets.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Dependencies](https://img.shields.io/badge/dependencies-none-brightgreen.svg)](#architecture)
[![Offline capable](https://img.shields.io/badge/offline-capable-brightgreen.svg)](#offline-behaviour)
[![PRs welcome](https://img.shields.io/badge/PRs-welcome-orange.svg)](CONTRIBUTING.md)

---

## Table of contents

- [Overview](#overview)
- [Key capabilities](#key-capabilities)
- [Getting started](#getting-started)
- [Usage](#usage)
- [Configuration reference](#configuration-reference)
- [Architecture](#architecture)
- [External services](#external-services)
- [Offline behaviour](#offline-behaviour)
- [Browser support](#browser-support)
- [Known limitations](#known-limitations)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [Security and privacy](#security-and-privacy)
- [License and attribution](#license-and-attribution)

---

## Overview

Cardboard Golem converts deck lists into PDF sheets containing playtest proxies at the official card dimension of 63 × 88 mm. It ships as one HTML file with no build pipeline, no runtime dependencies, and no server component.

### Design principles

| Principle | Implementation |
|---|---|
| **Dimensional accuracy is non-negotiable** | A hand-written PDF generator places every card at exact point coordinates. The `MediaBox` is authoritative, eliminating browser print-scaling as a failure mode. |
| **The single file is a feature** | No bundler, no CDN, no webfonts. The application is auditable in one read and deployable by copying a file. |
| **Degrade, never fail** | Every network path has a fallback ending in a printable placeholder. An unresolvable card never prevents export. |
| **Source fidelity is preserved** | Images are embedded at their native resolution without resampling. Rescaling is delegated to the printer's raster image processor. |

---

## Key capabilities

### Deck list ingestion
- Parses Arena, Moxfield, MTGO, and plain-text formats
- Tolerates bullet characters, tab-delimited spreadsheet exports, trailing quantity notation, byte-order marks, foil markers, and section headers
- Defaults absent quantities to `1` rather than rejecting the line
- Preserves split-card names (`Fire // Ice`) while honouring `//` line comments

### Card resolution
A five-tier resolution ladder, executed in order until a match is found:

| Tier | Method | Purpose |
|---|---|---|
| 0 | IndexedDB cache | Zero-latency repeat lookups |
| 1 | `POST /cards/collection` | Batched resolution, 75 identifiers per request |
| 2 | `GET /cards/named?exact=` | Preflight-free canonical match |
| 3 | `GET /cards/named?fuzzy=` | Typo and partial-name tolerance |
| 4 | `GET /cards/search` | Full-text fallback |
| — | Placeholder | Named outline; sheet remains printable |

Tier 1 requires a CORS preflight and self-disables permanently on first failure, ensuring `file://` deployments fall through to the GET-only path without data loss.

### Output generation
- Hand-implemented PDF 1.4 writer with cross-reference table construction
- JPEG embedding via `/DCTDecode` (pass-through, no re-compression of the container)
- Lossless PNG embedding via `/FlateDecode` with PNG predictor 15
- Configurable crop marks, full-bleed cut lines, inter-card gaps, and safe margins
- Optional 100 mm calibration scale on the first sheet for print-scaling verification

### Supplementary features
- Token detection via Scryfall `all_parts`, resolved by canonical ID
- Market pricing (EUR/USD) with self-accumulating trend history capped at 30 daily snapshots
- Dual art sourcing from Scryfall and MPC Autofill, presented in a unified selector
- Scan-quality warnings derived from the `image_status` field
- Canvas-readability pre-flight for third-party images, surfaced before export

---

## Getting started

### Prerequisites

A modern browser. Nothing else.

### Installation

```bash
git clone https://github.com/you/cardboard-golem.git
cd cardboard-golem
```

### Running

**Option A — direct** (fastest)

Open `index.html` in a browser.

**Option B — local HTTP server** (recommended)

```bash
python3 -m http.server 8000
```

Then navigate to `http://localhost:8000`.

> **Note on origins.** Loading via `file://` assigns the page a `null` origin. Some browser and network configurations reject cross-origin requests from `null`, disabling the batched resolution endpoint and potentially blocking image caching. The application detects this condition and surfaces remediation guidance. Serving over `localhost` avoids it entirely and materially improves bulk-import performance.

---

## Usage

1. **Provide a deck list** — paste text, drop a `.txt`/`.dec`/`.dek` file, or use the incremental card search.
2. **Review resolution** — inspect flagged rows for fuzzy matches, low-resolution scans, or unresolved entries.
3. **Adjust the sheet** — select paper size, cut guides, and image quality. The preview reflects final output geometry.
4. **Export** — generate the PDF and print with scaling disabled (*Actual size* / 100 %).

### Print settings

| Setting | Required value |
|---|---|
| Scaling | Actual size / 100 % — **not** "Fit to page" |
| Paper size | Must match the size selected in-app |
| Resolution | Printer native (typically 600 dpi) |

Higher printer resolutions do not improve output. The source image is 745 px, which corresponds to exactly 300 dpi at 63 mm width; interpolation beyond that adds no information.

### Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + Enter` | Import deck list |
| `Ctrl/Cmd + P` | Export PDF (intercepts browser print) |
| `↑` / `↓` | Navigate search suggestions |
| `Esc` | Dismiss suggestions or modal |

---

## Configuration reference

All settings persist to `localStorage` and are exposed through the interface.

| Key | Type | Default | Description |
|---|---|---|---|
| `paper` | enum | `A4` | `A4`, `LETTER`, `A4L`, `LETTERL` |
| `quality` | enum | `png` | Source image tier: `png`, `large`, `border_crop`, `normal` |
| `cardW` / `cardH` | number (mm) | `63` / `88` | Card dimensions; adjust for printer calibration |
| `gap` | number (mm) | `0` | Inter-card spacing |
| `margin` | number (mm) | `6.5` | Minimum distance from sheet edge |
| `guides` | enum | `marks` | `marks`, `lines`, `none` |
| `enc` | enum | `jpeg` | PDF image codec: `jpeg` (q95) or `png` (lossless) |
| `hinta` | enum | `eur` | Price currency: `eur`, `usd`, `off` |
| `ruler` | boolean | `true` | Render 100 mm calibration scale on sheet 1 |
| `dfc` | boolean | `true` | Emit both faces of double-faced cards |
| `cacheImgs` | boolean | `true` | Persist image blobs to IndexedDB |

---

## Architecture

The application is organised into numbered sections within a single file, each with a defined responsibility:

| Section | Responsibility |
|---|---|
| § 1 | Deck list parsing and normalisation |
| § 2 | IndexedDB persistence layer |
| § 3 | Card resolution, image tier selection, external sources |
| § 4 | Application state and settings persistence |
| § 5 | Card list rendering |
| § 6 | Sheet geometry, unit flattening, preview rendering |
| § 7 | User-facing actions |
| § 8 | Event binding |
| § 9 | PDF generation |

A single `laskeMitat()` (geometry) function supplies both the on-screen preview and the PDF writer, structurally guaranteeing that the preview cannot misrepresent output.

### Code conventions

Internal identifiers and comments are written in Finnish; all user-facing strings are English. DOM identifiers, CSS class names, and external API parameters remain in English as they constitute technical interfaces rather than domain vocabulary.

### Rate limiting

Outbound requests are throttled to one per 110 ms, remaining within Scryfall's published guidance of 50–100 ms between calls and well under their 10 requests/second ceiling.

---

## External services

| Service | Endpoints | Authentication | Failure mode |
|---|---|---|---|
| Scryfall | `/cards/collection`, `/cards/named`, `/cards/search`, `/cards/autocomplete`, `/cards/:id` | None | Falls back through resolution ladder; cached data remains available |
| MPC Autofill | `/2/searchResults/`, `/2/cards/` | None | Returns empty result set; Scryfall results unaffected |

The MPC Autofill integration targets an undocumented endpoint. It is defensively wrapped and degrades to an empty list on any schema or availability change.

---

## Offline behaviour

| Asset | Store | Persistence |
|---|---|---|
| Card metadata | IndexedDB `kortit` | Indefinite, user-clearable |
| Card images | IndexedDB `kuvat` | Indefinite, user-clearable |
| Settings and deck text | `localStorage` | Indefinite |

Once a deck has been resolved and its images cached, subsequent sessions require no network access. The application performs a connectivity probe on import and transparently switches to cache-only operation when offline.

---

## Browser support

| Browser | Minimum | Notes |
|---|---|---|
| Chrome / Edge | 90+ | Full support |
| Firefox | 90+ | Full support |
| Safari | 16+ | `100dvh` used in mobile layout |

Requires `IndexedDB`, `fetch`, `canvas.toBlob`, and `TextEncoder`.

---

## Known limitations

| Limitation | Cause | Mitigation |
|---|---|---|
| 300 dpi effective ceiling | Largest available source image is 745 px | Documented in-app; higher printer DPI provides no benefit |
| Price trends unavailable on first use | No keyless historical pricing API exists | Snapshots accumulate from first run; arrows appear on second day |
| MPC Autofill images may not embed | Google Drive CORS policy blocks canvas reads | Pre-flight check warns before export; affected cards render as outlines |
| Manual additions cleared on re-import | List state is derived from the deck list textarea | Surfaced in status messaging |

---

## Roadmap

- [ ] Persist tokens and manually added cards across re-imports
- [ ] Card back printing for duplex output
- [ ] Per-printer calibration offset profiles
- [ ] LRU eviction for the image cache
- [ ] Bleed support for commercial print services

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a pull request.

**Non-negotiable constraints:**

1. The application must remain a single file with no runtime dependencies.
2. Any change affecting layout or PDF generation must be verified against physical output — the 100 mm calibration scale must measure 100 mm on paper.
3. User-facing strings are English; internal identifiers and comments are Finnish.

---

## Security and privacy

- **No telemetry.** The application transmits nothing except card lookups to the services listed above.
- **No accounts, no tracking, no analytics.**
- **All user data is local.** Deck lists, settings, and cached images never leave the device.
- **No secrets in source.** The application requires no API keys and stores no credentials.

---

## License and attribution

### Software

Licensed under the [MIT License](LICENSE).

### Card content

Card names, images, artwork, mana symbols, and Oracle text are the property of Wizards of the Coast LLC and are **not** covered by the MIT license.

> Cardboard Golem is unofficial Fan Content permitted under the Wizards of the Coast Fan Content Policy. Not approved or endorsed by Wizards. Portions of the materials used are property of Wizards of the Coast. © Wizards of the Coast LLC.

### Data attribution

Card data and imagery are sourced from [Scryfall](https://scryfall.com). This project is not produced by, endorsed by, supported by, or affiliated with Scryfall, LLC.

In accordance with Scryfall's API guidelines, this project does not paywall access to Scryfall data, does not require accounts or subscriptions, and surfaces artist and source attribution alongside every card image displayed.

### Intended use

Proxies generated by this software are intended for personal playtesting and casual play. They are not tournament legal. Producing proxies for sale is prohibited under the Fan Content Policy and constitutes copyright infringement.
