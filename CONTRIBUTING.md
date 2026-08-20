# Contributing to Cardboard Golem

Thanks for considering a contribution. This document covers the constraints that make this project what it is, and the checks that keep it working.

---

## The three constraints

These are not style preferences. Breaking any of them changes what the project is.

### 1. One file, zero runtime dependencies

Everything lives in `index.html`. No CDN links, no webfonts, no npm packages, no build step.

The reason is concrete: this tool must work on a laptop with no internet connection, five years from now, when the CDN you linked has changed its URL scheme. If a feature genuinely cannot be built without a library, open an issue to discuss it before writing code.

### 2. The millimetres are sacred

A proxy printed at 96 % scale is unusable — it won't sleeve alongside real cards. Any change touching sheet geometry or the PDF writer must be verified against **physical paper**, not just a screen preview.

The verification is built in: enable the 100 mm scale bar, export, print at Actual size, and measure it with a ruler. If it isn't 100 mm, something regressed.

### 3. Degrade, never crash

Every network call has a fallback. Every fallback ends somewhere printable. A card that cannot be found becomes a named outline; it never blocks the export or throws an unhandled error.

When adding a new external call, wrap it so that total failure produces an empty result rather than an exception.

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
- Sections are numbered (`§ 1` … `§ 9`) with a header comment explaining the section's responsibility
- Comments explain **why**, not what. `// throttle to 110ms` is noise; `// max ~9 req/s so we don't get rate limited mid-import` is useful
- No frameworks, no transpilation. Plain ES2020 that runs directly in a browser

---

## Before you open a pull request

Run through this list. Most rejected PRs fail on the first or last item.

- [ ] **Physical print test** if you touched layout or PDF code — export, print, measure the 100 mm bar
- [ ] **Mobile check** at 375 px width — every interactive element still reachable, no horizontal scroll, no input smaller than 16 px font
- [ ] **Desktop check** at 1440 px width
- [ ] **Offline check** — DevTools → Network → Offline. The app should load and work from cache
- [ ] **PDF opens in a real reader** — Acrobat, Preview, or Firefox's viewer. Browser print preview is not sufficient
- [ ] **No new dependencies** — no `<script src>`, no `@import url()`, no `fetch` to a CDN
- [ ] **Console is clean** — no errors, no warnings you introduced

---

## Reporting bugs

A useful bug report includes:

1. **What you did** — ideally the exact deck list line that triggered it
2. **What you expected**
3. **What happened instead**
4. **Browser and version**, and whether you opened the file directly or served it over `localhost`
5. **Console output**, if there is any

That last point about `file://` vs `localhost` resolves a surprising share of reports on its own.

### Print quality issues

Before reporting blurry output, please check:

- Was printer scaling set to **Actual size** / 100 %?
- Does the 100 mm bar on sheet 1 measure 100 mm?
- Does the card row show a **low-res scan** badge? If so, that printing genuinely has no high-resolution source — try the Art button for a different printing.
- Was PDF encoding set to JPEG or PNG? PNG is lossless and noticeably sharper on small text.

---

## Suggesting features

Open an issue first for anything non-trivial. Useful proposals explain:

- The workflow problem it solves
- Why it can't be solved with existing settings
- How it survives all three constraints above

Features most likely to be accepted are ones that make the printed output more accurate or the import more forgiving. Features least likely are ones that add visual complexity to the interface — this is a utility, and restraint is part of the design.

---

## Code of conduct

Be decent. Assume good faith. Critique code, not people.

---

## License

By contributing, you agree that your contributions will be licensed under the [MIT License](LICENSE) that covers this project.
