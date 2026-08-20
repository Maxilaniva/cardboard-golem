# Contributing to Cardboard Golem

Thanks for considering a contribution. This document covers the constraints that make this project what it is, and the checks that keep it working.

---

## The three constraints

These are not style preferences. Breaking any of them changes what the project is.

### 1. One file

Everything lives in `index.html`. No build step, no bundler, no module graph.

The reason is concrete: this tool must work on a laptop with no internet connection, five years from now. A single file has no supply chain to rot.

### 2. Nothing loads before it is needed

The core application must open and function with the network unplugged. That means no CDN script tags, no `@import url()`, no webfonts, no analytics.

**There is one bounded exception**, and it defines the rule for any future one. Camera OCR loads Tesseract.js from jsDelivr — but only when the user actually presses Scan, in a feature that already requires a camera and a live connection, and it degrades to manual entry if the fetch fails.

A new optional dependency is acceptable only if it satisfies all three:

- [ ] It loads **on demand**, never at page load
- [ ] It serves a feature that **cannot work offline anyway**
- [ ] Its failure degrades to a working fallback, **not** an error state

If a proposed dependency fails any of these, open an issue to discuss before writing code.

### 3. The millimetres are sacred

A proxy printed at 96 % scale is unusable — it won't sleeve alongside real cards. Any change touching sheet geometry or the PDF writer must be verified against **physical paper**, not just a screen preview.

The verification is built in: enable the 100 mm scale bar, export, print at Actual size, and measure it with a ruler. If it isn't 100 mm, something regressed.

---

## Code conventions

| Element | Language | Example |
|---|---|---|
| Internal identifiers | Finnish | `piirraArkit()`, `TILA.kortit` |
| Code comments | Finnish, conversational | `// Ekaks katotaan löytyykö välimuistista` |
| User-facing UI strings | English | `"Import list"`, `"No cards yet."` |
| DOM ids, CSS classes | English | `#cardList`, `.sheetwrap` |
| External API parameters | As the API defines them | `unique=prints`, `image_status` |

The split is deliberate: domain vocabulary is Finnish because that's the author's working language, but anything that is a technical interface or a user sees stays English.

Other conventions:
- Sections are numbered (`§ 0b` … `§ 11`) with a header comment explaining the section's responsibility
- Comments explain **why**, not what. `// throttle to 110ms` is noise; `// max ~9 req/s so we don't get rate limited mid-import` is useful
- Plain ES2020 that runs directly in a browser — no transpilation

---

## Before you open a pull request

Run through this list. Most rejected PRs fail on the first or last item.

**Always**
- [ ] **Offline check** — DevTools → Network → Offline. The app loads and works from cache
- [ ] **Mobile check** at 375 px width — every interactive element reachable, no horizontal scroll, no input under 16 px font
- [ ] **Desktop check** at 1440 px width
- [ ] **Console is clean** — no errors, no warnings you introduced
- [ ] **No new page-load dependencies** — no `<script src>`, no `@import url()`

**If you touched layout or the PDF writer**
- [ ] **Physical print test** — export, print at Actual size, measure the 100 mm bar
- [ ] **PDF opens in a real reader** — Acrobat, Preview, or Firefox's viewer. Browser print preview is not sufficient

**If you touched card resolution**
- [ ] Import a 100-card list and confirm it completes
- [ ] Simulate a `429` response and confirm the app backs off rather than continuing to request
- [ ] Confirm a genuinely nonexistent card is still marked missing after the full ladder

**If you touched the scanner**
- [ ] Test over `localhost` — `file://` cannot use the camera at all
- [ ] Deny camera permission and confirm the message names the actual cause
- [ ] Block jsDelivr and confirm it falls back to the manual crop rather than erroring

---

## A note on verification

Syntax checks are not behaviour checks. A missing function reference passes `node --check` and fails at runtime — this has already bitten this project once.

Where practical, extract the logic you changed into a small harness and test it against real inputs. The geometry, sharpness scoring, OCR cleanup, price trend, and resolution ladder are all pure enough to test this way, and several real bugs have been caught doing exactly that.

---

## Reporting bugs

A useful bug report includes:

1. **What you did** — ideally the exact deck list line that triggered it
2. **What you expected**
3. **What happened instead**
4. **Browser and version**, and whether you opened the file directly or served it over `localhost`
5. **Console output**, if there is any

That fourth point resolves a surprising share of reports on its own.

### Print quality issues

Before reporting blurry output, please check:

- Was printer scaling set to **Actual size** / 100 %?
- Does the 100 mm bar on sheet 1 measure 100 mm?
- Does the card row show a **low-res scan** badge? If so, that printing genuinely has no high-resolution source — try the Art button for a different printing.
- Was PDF encoding set to JPEG or PNG? PNG is lossless and noticeably sharper on small text.

### Scanner issues

- Are you on `https://` or `localhost`? Nothing else can access a camera.
- Does the message mention Shields, permissions, or another app using the camera? Each has a different fix.
- Foil cards under direct light are the known worst case; tilting the card usually helps more than any setting.

---

## Suggesting features

Open an issue first for anything non-trivial. Useful proposals explain:

- The workflow problem it solves
- Why it can't be solved with existing settings
- How it survives all three constraints above

Features most likely to be accepted make the printed output more accurate or the import more forgiving. Features least likely add visual complexity to the interface — this is a utility, and restraint is part of the design.

---

## Code of conduct

Be decent. Assume good faith. Critique code, not people.

---

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE) that covers this project.
