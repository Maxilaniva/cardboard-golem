# Cardboard Golem

A single-file web application that renders Magic: The Gathering deck lists into dimensionally accurate, print-ready proxy sheets.

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Core dependencies](https://img.shields.io/badge/core%20dependencies-none-brightgreen.svg)](#dependency-policy)
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
- [Dependency policy](#dependency-policy)
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

Cardboard Golem converts deck lists into PDF sheets containing playtest proxies at the official card dimension of 63 × 88 mm. It ships as one HTML file with no build pipeline and no server component.

### Design principles

| Principle | Implementation |
|---|---|
| **Dimensional accuracy is non-negotiable** | A hand-written PDF generator places every card at exact point coordinates. The `MediaBox` is authoritative, eliminating browser print-scaling as a failure mode. |
| **The single file is a feature** | No bundler, no framework, no webfonts. The application is auditable in one read and deployable by copying a file. |
| **Nothing loads before it is needed** | The core application makes zero third-party requests. Optional subsystems fetch on demand and degrade gracefully on failure. |
| **Degrade, never fail** | Every network path has a fallback ending in a printable placeholder. An unresolvable card never prevents export. |
| **Source fidelity is preserved** | Images are embedded at native resolution without resampling. Rescaling is delegated to the printer's raster image processor. |

---

## Key capabilities

### Deck list ingestion
- Parses Arena, Moxfield, MTGO, and plain-text formats
- Tolerates bullet characters, tab-delimited spreadsheet exports, trailing quantity notation, byte-order marks, foil markers, and section headers
- Defaults absent quantities to `1` rather than rejecting the line
- Preserves split-card names (`Fire // Ice`) while honouring `//` line comments
- Accepts file input via drag-and-drop or file picker (`.txt`, `.dec`, `.dek`)

### Card resolution

A five-tier ladder, executed in order until a match is found:

| Tier | Method | Purpose |
|---|---|---|
| 0 | IndexedDB cache | Zero-latency repeat lookups |
| 1 | `POST /cards/collection` | Batched resolution, 75 identifiers per request |
| 2 | `GET /cards/named?exact=` | Preflight-free canonical match |
| 3 | `GET /cards/named?fuzzy=` | Typo and partial-name tolerance |
| 4 | `GET /cards/search` | Full-text fallback |
| — | Placeholder | Named outline; sheet remains printable |

Tier 1 requires a CORS preflight and self-disables permanently on first failure, ensuring `file://` deployments fall through to the GET-only path without data loss.

**Error classification.** HTTP responses are triaged into three categories with distinct handling: `404` advances to the next tier; `429` and `5xx` trigger exponential backoff (600/1500/3500 ms, honouring `Retry-After`) and, on exhaustion, abort the run cleanly; network failures advance to the next tier. Cards not reached during an aborted run retain pending state rather than being marked missing, allowing a retry to resume precisely.

### Interactive card entry
- Incremental search backed by `GET /cards/autocomplete`, debounced at 180 ms with a session-scoped query cache and stale-response discarding
- Camera scanning in batch mode (see below)

### Camera scanning

| Stage | Implementation |
|---|---|
| Capture | Rear camera at 720p with a constraint-relaxation ladder; burst of 5 frames over ~1 s |
| Frame selection | Laplacian variance scoring; sharpest frame retained, blurred captures rejected pre-OCR |
| Preprocessing | ROI crop to the card's name band (68 % width, excluding mana cost), greyscale conversion, contrast expansion, 3× upscale |
| Recognition | Tesseract.js in a Web Worker instantiated from a Blob URL, constrained to a letter/punctuation whitelist and single-line page segmentation |
| Matching | OCR output routed through autocomplete, then prefix truncation, then fuzzy lookup |
| Confirmation | Auto-accept requires OCR confidence ≥ 78 % and an unambiguous single match; otherwise presents three candidates |
| Batch flow | Accepted cards return immediately to the viewfinder with a running thumbnail strip |

### Card information
- Oracle text rendering with mana symbol substitution and dimmed reminder text
- Keyword glossary limited to keywords present in the card's structured `keywords` array; unrecognised keywords are omitted rather than inferred
- Official rulings fetched lazily on first open and cached
- Token detection via `all_parts`, resolved by canonical Scryfall ID

### Output generation
- Hand-implemented PDF 1.4 writer with cross-reference table construction
- JPEG embedding via `/DCTDecode`; lossless PNG via `/FlateDecode` with PNG predictor 15
- Configurable crop marks, cut lines, inter-card gaps, and safe margins
- Optional 100 mm calibration scale on the first sheet

### Supplementary
- Market pricing (EUR/USD) with self-accumulating trend history, capped at 30 daily snapshots per card
- Dual art sourcing from Scryfall and MPC Autofill in a unified selector, with canvas-readability pre-flight for third-party images
- Scan-quality warnings derived from `image_status`
- Persistent storage requested via `navigator.storage.persist()` on first import

---

## Getting started

### Prerequisites

A modern browser. Nothing else.

### Installation

```bash
git clone https://github.com/Maxilaniva/cardboard-golem.git
cd cardboard-golem
```

### Running

**Option A — direct**

Open `index.html` in a browser.

**Option B — local HTTP server** (recommended)

```bash
python3 -m http.server 8000
```

Then navigate to `http://localhost:8000`.

> **Note on secure contexts.** Loading via `file://` assigns the page a `null` origin. This disables the batched resolution endpoint (increasing request volume by up to 50×), may block image caching, and prevents camera access entirely. A LAN address such as `http://192.168.1.50:8000` is likewise not a secure context and cannot use the camera; only `https://` and `localhost` qualify. The application detects each condition and surfaces targeted remediation guidance.

---

## Usage

1. **Provide cards** — paste text, drop a file, use incremental search, or scan physical cards.
2. **Review resolution** — inspect flagged rows for fuzzy matches, low-resolution scans, or unresolved entries.
3. **Adjust the sheet** — select paper size, cut guides, and image quality. The preview reflects final output geometry.
4. **Export** — generate the PDF and print with scaling disabled.

### Print settings

| Setting | Required value |
|---|---|
| Scaling | Actual size / 100 % — **not** "Fit to page" |
| Paper size | Must match the size selected in-app |
| Resolution | Printer native (typically 600 dpi) |

Higher printer resolutions do not improve output. The source image is 745 px, corresponding to exactly 300 dpi at 63 mm width; interpolation beyond that adds no information.

### Keyboard shortcuts

| Shortcut | Action |
|---|---|
| `Ctrl/Cmd + Enter` | Import deck list |
| `Ctrl/Cmd + P` | Export PDF (intercepts browser print) |
| `↑` / `↓` | Navigate search suggestions |
| `Esc` | Dismiss suggestions, then modal |

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

The application is organised into numbered sections within a single file:

| Section | Responsibility |
|---|---|
| § 0b | Work-state tracking (concurrent operation counter driving the activity indicator) |
| § 0c | Logo animation and loading-state styling |
| § 1 | Deck list parsing and normalisation |
| § 2 | IndexedDB persistence layer |
| § 3 | Card resolution, image tier selection, external sources, price history |
| § 4 | Application state and settings persistence |
| § 5 | Card list rendering |
| § 6 | Sheet geometry, unit flattening, preview rendering |
| § 7 | User-facing actions |
| § 8 | Event binding |
| § 9 | PDF generation |
| § 10 | Card rules, keyword glossary, rulings |
| § 11 | Camera scanning, OCR worker, batch flow |

A single `laskeMitat()` (geometry) function supplies both the on-screen preview and the PDF writer, structurally guaranteeing that the preview cannot misrepresent output.

### Code conventions

Internal identifiers and comments are written in Finnish; all user-facing strings are English. DOM identifiers, CSS class names, and external API parameters remain in English as they constitute technical interfaces rather than domain vocabulary.

### Rate limiting

Outbound requests are throttled to one per 110 ms, within Scryfall's published guidance of 50–100 ms between calls and well under their 10 requests/second ceiling. Server-side rejection triggers backoff rather than continued traversal of the resolution ladder.

---

## Dependency policy

The core application has **no runtime dependencies**: no CDN, no webfont, no framework, no package manager.

One optional subsystem is exempt. **Camera OCR loads Tesseract.js (Apache-2.0) from jsDelivr on first scan invocation** — never at page load. The exemption is bounded by three properties:

1. The feature already requires a camera, a secure context, and network connectivity; it cannot function offline under any implementation.
2. No other subsystem references it. Import, export, printing, and caching are unaffected by its absence.
3. Load failure degrades to manual entry with an enlarged name crop, not to an error state.

To eliminate the dependency entirely, remove the `OCR_CDN` constant and `kaynnistaWorker()` in § 11; capture and manual transcription remain functional.

---

## External services

| Service | Endpoints | Auth | Failure mode |
|---|---|---|---|
| Scryfall | `/cards/collection`, `/cards/named`, `/cards/search`, `/cards/autocomplete`, `/cards/:id`, `/cards/:id/rulings` | None | Backoff, then ladder fallback; cached data remains available |
| MPC Autofill | `/2/searchResults/`, `/2/cards/` | None | Returns empty result set; Scryfall results unaffected |
| jsDelivr | `tesseract.js@5.1.1` | None | Scanner degrades to manual transcription |

The MPC Autofill integration targets an undocumented endpoint. It is defensively wrapped and degrades to an empty list on any schema or availability change.

---

## Offline behaviour

| Asset | Store | Persistence |
|---|---|---|
| Card metadata | IndexedDB `kortit` | Indefinite, user-clearable |
| Card images | IndexedDB `kuvat` | Indefinite, user-clearable |
| Rulings | IndexedDB `kortit` (`saannot:` prefix) | Indefinite |
| Settings and deck text | `localStorage` | Indefinite |

The application requests persistent storage on first import to reduce eviction risk under storage pressure. A connectivity probe on import triggers transparent cache-only operation when offline.

Browser settings that clear site data on close override this and will discard the cache between sessions.

---

## Browser support

| Browser | Minimum | Notes |
|---|---|---|
| Chrome / Edge | 90+ | Full support |
| Firefox | 90+ | Full support |
| Safari | 16+ | `100dvh` used in mobile layout; `navigator.permissions` lacks a `camera` descriptor, so permission-state diagnostics degrade to generic guidance |
| Brave | Any | Shields may block jsDelivr and Scryfall; the app detects Brave and names the Shields control specifically |

Requires `IndexedDB`, `fetch`, `canvas.toBlob`, `TextEncoder`, and `Worker`. Camera scanning additionally requires `getUserMedia` in a secure context.

---

## Known limitations

| Limitation | Cause | Mitigation |
|---|---|---|
| 300 dpi effective ceiling | Largest available source image is 745 px | Documented in-app; higher printer DPI provides no benefit |
| Camera unavailable on `file://` and LAN addresses | Secure-context requirement for `getUserMedia` | Detected with targeted guidance toward `localhost` |
| OCR accuracy approximately 70–85 % first attempt | Stylised typeface, variable lighting, foil glare | Burst capture with sharpness selection; three-candidate confirmation; manual fallback |
| Price trends unavailable on first use | No keyless historical pricing API exists | Snapshots accumulate from first run |
| MPC Autofill images may not embed | Google Drive CORS policy blocks canvas reads | Pre-flight check warns before export; affected cards render as outlines |
| Manual additions, scans, and tokens cleared on re-import | List state is derived from the deck list textarea | Surfaced in status messaging |

---

## Roadmap

- [ ] Persist tokens, scans, and manually added cards across re-imports
- [ ] Card back printing for duplex output
- [ ] Per-printer calibration offset profiles
- [ ] LRU eviction for the image cache
- [ ] Perspective correction prior to OCR
- [ ] Bleed support for commercial print services

---

## Contributing

Contributions are welcome. Please read [CONTRIBUTING.md](CONTRIBUTING.md) before submitting a pull request.

**Non-negotiable constraints:**

1. The application must remain a single file.
2. Nothing may load before it is needed; the app must open and operate with the network unavailable.
3. Any change affecting layout or PDF generation must be verified against physical output — the 100 mm calibration scale must measure 100 mm on paper.
4. User-facing strings are English; internal identifiers and comments are Finnish.

---

## Security and privacy

- **No telemetry.** The application transmits nothing except card lookups to the services listed above.
- **No accounts, no tracking, no analytics.**
- **All user data is local.** Deck lists, settings, scanned results, and cached images never leave the device.
- **Camera frames are never transmitted.** OCR executes entirely in-browser within a Web Worker; captured images are discarded after processing.
- **No secrets in source.** The application requires no API keys and stores no credentials.

---

## License and attribution

### Software

Licensed under the [MIT License](LICENSE).

### Third-party components

Tesseract.js (Apache-2.0) is loaded at runtime by the optional camera OCR subsystem. It is not vendored into this repository.

### Card content

Card names, images, artwork, mana symbols, and Oracle text are the property of Wizards of the Coast LLC and are **not** covered by the MIT license.

> Cardboard Golem is unofficial Fan Content permitted under the Wizards of the Coast Fan Content Policy. Not approved or endorsed by Wizards. Portions of the materials used are property of Wizards of the Coast. © Wizards of the Coast LLC.

### Data attribution

Card data and imagery are sourced from [Scryfall](https://scryfall.com). This project is not produced by, endorsed by, supported by, or affiliated with Scryfall, LLC.

In accordance with Scryfall's API guidelines, this project does not paywall access to Scryfall data, does not require accounts or subscriptions, and surfaces artist and source attribution alongside every card image displayed.

### Intended use

Proxies generated by this software are intended for personal playtesting and casual play. They are not tournament legal. Producing proxies for sale is prohibited under the Fan Content Policy and constitutes copyright infringement.
