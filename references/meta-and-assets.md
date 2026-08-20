# Head template, favicons, and OG preview image

Read this when polishing a site for public sharing: setting up the full `<head>` (title, description, OG, Twitter, favicons, drewhoover.com chrome) and generating the favicon set + OG preview image. This is the "make it look right when pasted into a chat" pass.

## The full `<head>` for a polished site

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />

    <title>{Compelling, scannable, ≤ 60 chars}</title>
    <meta name="description" content="{1–2 sentences, what + who-for. Used by search + when no OG description.}" />

    <!-- Favicons. SVG for modern, PNG fallbacks for Safari/iOS. -->
    <link rel="icon" type="image/svg+xml" href="/<slug>/favicon.svg" />
    <link rel="icon" type="image/png" sizes="32x32"  href="/<slug>/favicon-32.png" />
    <link rel="icon" type="image/png" sizes="192x192" href="/<slug>/favicon-192.png" />
    <link rel="apple-touch-icon" sizes="180x180"      href="/<slug>/apple-touch-icon.png" />

    <!-- Open Graph (Facebook, Reddit, Bluesky, LinkedIn) -->
    <meta property="og:type" content="website" />
    <meta property="og:site_name" content="{Short site name}" />
    <meta property="og:title" content="{Same as <title> or richer}" />
    <meta property="og:description" content="{Engaging hook, ~200 chars}" />
    <meta property="og:url" content="https://drewhoover.com/<slug>/" />
    <meta property="og:image" content="https://drewhoover.com/<slug>/og.png" />
    <meta property="og:image:width" content="1200" />
    <meta property="og:image:height" content="630" />
    <meta property="og:image:alt" content="{Describe the preview image for screen readers.}" />

    <!-- Twitter / X -->
    <meta name="twitter:card" content="summary_large_image" />
    <meta name="twitter:title" content="{Same as og:title}" />
    <meta name="twitter:description" content="{Same as og:description}" />
    <meta name="twitter:image" content="https://drewhoover.com/<slug>/og.png" />

    <!-- drewhoover.com cross-site chrome -->
    <script src="https://drewhoover.com/embed/back-bar.js" async></script>
    <script src="https://drewhoover.com/embed/giscus.js" async></script>
  </head>
  <body>
    <div id="root"></div>
    <div id="comments"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

Anchor `<div id="comments">` directly after the React root so giscus has a place to attach inside SPA layouts. The two scripts (`back-bar.js`, `giscus.js`) are the drop-in cross-site chrome from the index repo — every project gets them.

Opt-out if needed via `data-dhv-back-bar="off"` or `data-dhv-giscus="off"` on `<html>` or `<body>`.

## Generate the favicon set and OG preview image

`sharp` rasterizes SVG templates into PNGs at build time on a developer machine; the outputs are committed and served as static assets. CI does not regenerate them.

### Favicon set (`scripts/gen-favicon.mjs`)

```js
import sharp from 'sharp'
import { writeFileSync, mkdirSync } from 'node:fs'
import { dirname, resolve } from 'node:path'
import { fileURLToPath } from 'node:url'

const outDir = resolve(dirname(fileURLToPath(import.meta.url)), '..', 'public')
mkdirSync(outDir, { recursive: true })

// Design rule: must read at 16x16 in a browser tab strip. Use a bold
// background color (not white), high-contrast foreground, no fine detail.
const svg = `<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 64 64">
  <rect width="64" height="64" rx="12" fill="#1f4e8c"/>
  <!-- ...one simple, evocative glyph... -->
</svg>`

writeFileSync(resolve(outDir, 'favicon.svg'), svg + '\n')
for (const { name, size } of [
  { name: 'favicon-32.png',       size: 32  },
  { name: 'favicon-192.png',      size: 192 },
  { name: 'apple-touch-icon.png', size: 180 },
]) {
  await sharp(Buffer.from(svg)).resize(size, size).png({ compressionLevel: 9 }).toFile(resolve(outDir, name))
}
```

### OG preview image (`scripts/gen-og.mjs`)

Same shape, output `public/og.png` at 1200×630, raster from a hand-coded SVG. Anatomy that works:
- Dark gradient background (`#11151c` → `#0a0d12`) so colored accents pop.
- Large title + subtitle on the left in a system / DM Sans-ish stack.
- A stylized representative visual on the right (a fake chart, a mini-grid, a sample of the data).
- Footer with the URL and a thin accent bar.

### Index-card cover (`public/card.png`, 1200×750)

**Do not reuse `og.png` as the drewhoover.com project-card cover.** `ProjectCard.astro` renders covers as `aspect-ratio: 8/5; object-fit: cover`, so a 1200×630 image (1.90) gets centre-cropped — the header comes off the top and the last rows off the bottom, and nothing warns you. Emit a second variant at 1200×750 (exactly 8:5) that survives intact, and copy it over:

```bash
cp public/card.png ../DrewHoo.github.io/public/projects/<slug>.png
```

Parameterise `gen-og.mjs` by size rather than duplicating the SVG, and give the taller variant more rows since it has the room. Assert the layout fits — e.g. `if (lastRow + 18 > cardBottom) throw` — because a row running off the edge into the footer is easy to ship and only visible if you actually open the PNG. Open it.

### Wire up the scripts

Add to `package.json`:

```json
"gen:og":      "node scripts/gen-og.mjs",
"gen:favicon": "node scripts/gen-favicon.mjs"
```

Run them once, commit the outputs (`public/favicon*.png`, `public/favicon.svg`, `public/apple-touch-icon.png`, `public/og.png`, `public/card.png`). Re-run after any rebrand — and remember the OG image often depicts the page's main visual, so a redesign that removes that visual leaves the card advertising a section that no longer exists.

### OG cache busting after a redesign

Reddit / X / Bluesky / LinkedIn cache OG fetches for hours to days. Once the new image is live, you may need to re-share the URL with a cache-busting query param or use the platform's "rescrape" tool (Facebook Sharing Debugger, Twitter Card Validator) to force a refresh.
