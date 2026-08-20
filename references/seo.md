# SEO for the sibling sites

Read this before announcing a site, or when one isn't turning up in search.

## The big one: prerender, or the page has no content

A Vite SPA deploys this:

```html
<body><div id="root"></div></body>
```

Every word — headings, table rows, the intro copy — exists only after React runs. Google *will* usually render JS, but it's a deferred second pass with no guarantee, and Bing, DuckDuckGo, social unfurlers and the LLM crawlers largely won't. A new site with no authority is exactly the case where the deferred pass doesn't reliably happen.

Check any site with:

```bash
curl -s https://drewhoover.com/<slug>/ | grep -c '<div id="root"></div>'
```

`1` means the page is empty to a crawler.

The fix is a post-build step, not a framework. Vite's own dev server can load the app for SSR, so this reuses the existing source with no separate SSR bundle:

```js
// scripts/prerender.mjs — runs after `vite build`
import { createServer } from 'vite'
import { renderToString } from 'react-dom/server'
import React from 'react'
import fs from 'node:fs'

const vite = await createServer({ server: { middlewareMode: true }, appType: 'custom', logLevel: 'warn' })
const { default: App } = await vite.ssrLoadModule('/src/App.jsx')
const html = renderToString(React.createElement(App))
await vite.close()

const out = 'dist/index.html'
let page = fs.readFileSync(out, 'utf8')
if (!page.includes('<div id="root"></div>')) throw new Error('prerender: no empty #root found')
fs.writeFileSync(out, page.replace('<div id="root"></div>', `<div id="root">${html}</div>`))
```

```json
"build": "vite build && node scripts/prerender.mjs"
```

Throw if the marker is missing rather than silently no-op — otherwise a Vite change quietly reverts you to an empty page.

Two things this requires of the app:

1. **`main.jsx` must `hydrateRoot`, not `createRoot`.** `createRoot` throws the prerendered DOM away.
2. **No `window` reads during render.** The usual offender is seeding state from the URL:

   ```js
   // breaks SSR, and hydrates into a mismatch when ?item= is set
   const [sel, setSel] = useState(itemFromUrl)

   // works: render null on the server, apply the URL right after mount
   const [sel, setSel] = useState(null)
   useEffect(() => setSel(itemFromUrl()), [])
   ```

Verify: no hydration warnings in the console, and the state still applies on a deep link.

## One `<h1>`, and make it name the subject

Card-styled pages often have no `<h1>` at all, or one that reads like a section label. If the visible design wants a small eyebrow above a big title, wrap both in a single `h1` so it reads as one heading:

```jsx
<h1 className="m-head">
  <span className="m-eyebrow">& Juliet</span>
  <span className="m-title">The Release Timeline</span>
</h1>
```

Renders identically (`display: block` on both spans); the heading now says what the page is about instead of "The Release Timeline".

## Title and description: pick a query you can win

These pages don't outrank Wikipedia, Billboard or an official site on the obvious head term. What they have is an angle nobody else bothered with. Target that.

- ✗ `The 29 Max Martin Hits Behind & Juliet` — competes on "& Juliet songs", which is lost before it starts.
- ✓ `& Juliet Songs in Release Order: All 29 Max Martin Originals` — release order and dates are the page's actual differentiator.

Keep the title under ~60 characters and the description under ~155. Search what people type before writing either, and check `site:drewhoover.com <slug>` to see whether the page is indexed at all.

## Canonical, JSON-LD, sitemap

Canonical in `index.html`:

```html
<link rel="canonical" href="https://drewhoover.com/<slug>/" />
```

JSON-LD: generate it in the prerender step from the same data the page renders, so it can't drift from what's on screen. `ItemList` of the rows, plus `WebPage` and `BreadcrumbList`, in one `@graph`. Only describe what's actually visible — structured data that doesn't match the page is a penalty, not a boost.

Sitemap and robots live in the **index repo**, not the project:

- `DrewHoo.github.io/public/robots.txt` carries `Sitemap: https://drewhoover.com/sitemap-index.xml`.
- `@astrojs/sitemap` emits `sitemap-index.xml` + `sitemap-0.xml`. **There is no `/sitemap.xml`** — probing that path and concluding the sitemap is missing is an easy wrong turn.
- The project sites are separate deployments, so Astro never sees them. `astro.config.mjs` harvests `liveUrl` from the project cards and passes them as `sitemap({ customPages })`. Without that, `/projects/<slug>/` (the card detail page) is listed while the actual site is in no sitemap anywhere.

A per-project `dist/sitemap.xml` is worth little on its own — one URL — but it's what Search Console wants submitted and it carries `lastmod`.

## The index card is part of SEO

A link from the (indexed) homepage is the best authority signal these projects get, so a stale card is a real cost, not just untidy. When a site is redesigned, update `src/content/projects/<slug>.md` in the index repo — blurb, `coverAlt`, `updated` — in the same pass. See `references/index-registration.md`.
