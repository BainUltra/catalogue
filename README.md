# 2026 Condensed Price List

Interactive bilingual (EN/FR) flipbook of the BainUltra 2026 Condensed Price List.
Hosted on GitHub Pages — shareable to dealers, partners, and showroom staff via a
single link that lands them on a language-picker, then opens the corresponding
flipbook with search, table of contents, thumbnails, page-flip animations,
fullscreen, and downloadable PDF.

**Live URL:** https://rfrechette00-source.github.io/2026-condensed-price-list/

## Structure

```
.
├── index.html                                  Landing page — language picker
├── bainultra-wordmark-dark.svg                 Logo for the dark-text landing card
├── en/
│   ├── index.html                              English flipbook (223 pages)
│   ├── bainultra-wordmark.svg                  White logo for the topbar
│   ├── BU_Condensed_Price_List_2026.pdf        Source EN PDF (also the Download button target)
│   └── pages/
│       └── page-001.jpg … page-223.jpg         150-DPI JPGs, ~190 KB each
├── fr/
│   ├── index.html                              French flipbook (223 pages, Quebec French UI)
│   ├── bainultra-wordmark.svg
│   ├── Liste_de_prix_condensee_2026_FR.pdf
│   └── pages/
│       └── page-001.jpg … page-223.jpg
└── .gitignore
```

Every URL works in isolation:
- `/` — landing card (EN / FR picker), keyboard shortcuts `E` or `1` → English, `F` or `2` → Français
- `/en/` — English flipbook
- `/fr/` — French flipbook
- `/?auto=1` — auto-redirects to the last language the user picked (stored in `localStorage`)

The flipbook in either language has a globe button (🌐 EN/FR) in the topbar that
returns to the picker.

## How to share

Paste this single URL anywhere — Slack, email, SharePoint, dealer portal:

```
https://rfrechette00-source.github.io/2026-condensed-price-list/
```

Recipients land on the language picker and click through. The URL bar shows
`rfrechette00-source.github.io`, not BainUltra-Online-Hub, so it stays
self-contained.

### SharePoint embed

To make the link appear as a SharePoint URL:
1. In your SharePoint site, create a new **Site Page**.
2. Add an **Embed** web part.
3. Paste the live URL above.
4. Publish. The SharePoint URL becomes the shareable link, with the flipbook
   rendered inline. Requires your tenant to allow embedding `github.io` (most
   tenants do by default).

## Updating for next year

When BainUltra publishes the 2027 price list, you (or future-you, or another
agent) can refresh both flipbooks in about 5 minutes per language:

### 1. Replace the source PDF

Drop the new PDF into `en/` (named `BU_Condensed_Price_List_2026.pdf`) or `fr/`
(named `Liste_de_prix_condensee_2026_FR.pdf`). Keep the filenames — the
Download button in each flipbook is hardcoded to these paths.

### 2. Wipe and re-render the page JPGs

```bash
cd en   # or cd fr
rm -rf pages && mkdir pages
pdftoppm -jpeg -jpegopt "quality=82,progressive=y,optimize=y" -r 150 \
  BU_Condensed_Price_List_2026.pdf pages/page
```

This produces `pages/page-001.jpg` through `pages/page-NNN.jpg` at 1275×1650
(~190 KB each). Same DPI used the first time around — keeps quality consistent.

### 3. Watch for wide-page anomalies

Earlier editions of the EN PDF embedded the Lunessa 6636 product sheet as a
single double-wide page (2× normal width, ~2550 px) **and duplicated it on
the next page** — pdftoppm renders each wide page as a single JPG, which then
gets squished into a half-width book leaf, displaying both halves of the
spread overlapping. The fix is to split each wide JPG into its two halves:

```bash
sips --cropToHeightWidth 1650 1275 --cropOffset 0    0 page-NNN.jpg --out left.jpg
sips --cropToHeightWidth 1650 1275 --cropOffset 0 1275 page-NNN.jpg --out right.jpg
mv left.jpg page-NNN.jpg
mv right.jpg page-(NNN+1).jpg
```

Audit script:
```bash
for f in pages/*.jpg; do
  w=$(sips -g pixelWidth "$f" | tail -1 | awk '{print $2}')
  [ "$w" -gt 1500 ] && echo "$f is $w px wide"
done
```

If nothing prints, the PDF is clean.

### 4. Update page counts and references

In `en/index.html` and `fr/index.html`, find:
- `const N = 223;` → set to your actual page count
- `<div class="page-ind" id="pgInd">1 <span>/ 223</span></div>` → same
- `<p>2026 Condensed Price List · English</p>` → bump year in titles, loader,
  topbar subtitle, and `<title>` tag

If pages have shifted, also update:
- `tocData` array — TOC item page numbers
- `modelPages` array — search index page numbers
- `<button class="bb" onclick="goTo(N)">` quick-link bar at the bottom

Use the extraction recipe in `references/` (no such folder exists by default;
the historical script lives in this repo's commit history) to regenerate the
model index by parsing the PDF text:

```bash
mkdir -p /tmp/pdftext
for p in $(seq 1 223); do
  pdftotext -layout -f $p -l $p en/BU_Condensed_Price_List_2026.pdf \
    /tmp/pdftext/p$(printf '%03d' $p).txt
done
# Then grep first non-blank line after "Product Sheet" on each page —
# that's the model name. See git log for the exact script.
```

### 5. Bump the cache-buster

In both `en/index.html` and `fr/index.html`, find:
```js
const VER = 'v1';
```
Bump to `v2`, `v3`, etc. Each browser sees URLs like `page-117.jpg?v2` as a
different cache key, so they re-fetch instead of serving stale wide pages.

### 6. Update the landing page

If page counts changed in `index.html`:
```html
<span class="lang-note">223 pages</span>
```

If a new year, also update:
```html
<div class="card-eyebrow">2026 Condensed Price List</div>
<div class="footer">BainUltra · Lévis, Québec · effective January 1, 2026</div>
```

### 7. Commit & push

```bash
git add .
git commit -m "Update to 2027 condensed price list"
git push origin main
```

GitHub Pages rebuilds in ~30 seconds. Hard-refresh (Cmd+Shift+R) to bypass
any local browser cache.

## Tech notes

- **No build step.** Everything is static HTML, CSS, and inline JS. Edit a file,
  push, done.
- **No JavaScript framework.** ~1000 lines per flipbook, all vanilla JS. Each
  page leaf is a CSS-transformed `<div>` with `rotateY()` animation.
- **Page loading.** Each flipbook preloads ±6/8 pages around the current spread.
  Initial load is ~10 images (~2 MB), then more on demand.
- **Aspect ratio.** `PAGE_RATIO = 0.7778` matches the 630×810 pt source PDF.
- **Mobile.** Touch swipe and pinch-to-zoom work; the bottom collection bar is
  horizontally scrollable.
- **Brand voice.** All UI text follows BainUltra brand voice rules: lowercase
  headlines, no "luxury" or "buy now" language, calm/confident tone, Quebec
  French in the FR version with "vous" form (B2B/dealer-appropriate).

## Browser support

Tested on:
- Safari 17+ (macOS, iOS)
- Chrome / Edge 120+
- Firefox 120+

Requires CSS `backdrop-filter`, `transform: rotateY()`, and `backface-visibility`
— all available in 95%+ of browsers since 2020.

## License

Internal BainUltra asset. Do not redistribute outside authorized channels.

---

*Maintained by Ryan Frechette · BainUltra · Built with Claude Code*
