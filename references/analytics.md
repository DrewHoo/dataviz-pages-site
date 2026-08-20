# Analytics for the sibling sites

Read this when a project site should report pageviews and structured events, or when a site that looks wired up is reporting nothing.

## The one thing to know first

**Each project site is its own GitHub Pages deployment. None of them inherit the index site's analytics.** `drewhoover.com/<slug>/` shares an origin with the index site, but not a bundle — so a project reports events only if that repo did something about it. In Aug 2026, five published projects (`and-juliet`, `space-rock`, `collegiate-championships`, `should-you-buy-an-all-time-high`, `cfb-all-time-records`) had never been wired up and had reported nothing since launch. Nothing in the UI hints at this; the site just looks fine and no data arrives.

If a site is missing from Mixpanel, check this before debugging anything clever.

## Preferred: the shared embed (no dependency)

`DrewHoo.github.io/public/embed/analytics.js` is a drop-in, matching the `back-bar.js` / `giscus.js` convention. One tag in the project's `index.html`:

```html
<!-- drewhoover.com cross-site chrome -->
<script src="https://drewhoover.com/embed/back-bar.js" async></script>
<script src="https://drewhoover.com/embed/giscus.js" async></script>
<script src="https://drewhoover.com/embed/analytics.js" async></script>
```

That's the whole integration. No `mixpanel-browser` dependency in the project repo, one place to fix bugs, and new projects get it by copying the chrome block they were already copying.

What it does:
- Auto-pageview on load **and** on every history change, so `replaceState` URL state (see `references/url-state.md`) counts each view.
- Registers a `site` super-property (first path segment, `index` at the root) so reports split project sites apart without pathname parsing.
- Same origin as drewhoover.com, so `distinct_id` carries over — a visit that starts on the homepage and continues into a project reads as one person.
- Exposes `window.dhAnalytics.track(name, props)`, queued until init, dropped silently if blocked.
- Opt out with `data-dhv-analytics="off"` on `<html>` or `<body>`.

Structured events from a project (optional-chained — it's third-party and blockable):

```js
window.dhAnalytics?.track('Song selected', { id, title, artist, year })
```

Pageviews answer "what URL did they land on"; structured events make "which item did they look at" a one-click query.

### Maintaining the embed

Source is `src/scripts/embed-analytics.js` in the index repo; the served file is generated and committed:

```bash
npm run gen:embed   # esbuild --bundle --minify --format=iife -> public/embed/analytics.js
```

**It bundles the `mixpanel-browser` npm package deliberately.** Do not "simplify" it to a `<script src="https://cdn.mxpnl.com/libs/mixpanel-2-latest.min.js">`. That build assumes Mixpanel's official bootstrap snippet has already put a `mixpanel` stub on `window`; without it every page logs

```
Mixpanel error: "mixpanel" object not initialized. Ensure you are using the
latest version of the Mixpanel JS Library along with the snippet we provide.
```

and sends nothing. Bundling also keeps the file same-origin, so no third-party CDN lands on every project page. It's ~420KB minified (~130KB over the wire) but `async`, so it never blocks paint.

## Alternative: per-repo module

Still fine for a project that wants analytics in its own bundle — e.g. one that needs `track()` during module init, before an async embed could have loaded. `npm i mixpanel-browser`, then `src/analytics.js`:

```js
const TOKEN = '1c6a0f45b8a5768185a8d9a2f4d65452'

let mp = null
const queue = []

if (typeof window !== 'undefined') {
  import('mixpanel-browser')          // dynamic: keeps it out of the critical chunk
    .then((m) => {
      mp = m.default
      mp.init(TOKEN, { track_pageview: 'url-with-path-and-query-string' })
      for (const args of queue) { try { mp.track(...args) } catch {} }
      queue.length = 0
    })
    .catch(() => { queue.length = 0 })  // adblocked; never break the app
}

export function track(name, props) {
  if (mp) { try { mp.track(name, props) } catch {} } else queue.push([name, props])
}
```

Import it for side effects in `main.jsx`. **Do not use both this and the embed on one page** — two `init` calls means every pageview counted twice. The embed guards on `window.dhAnalytics`, which won't catch a bundled copy.

## The token

`1c6a0f45b8a5768185a8d9a2f4d65452` — shared across all the sites, public by design (write-only ingestion, no PII), safe to commit.

## Verifying it actually works

Analytics failing silently is the whole hazard: every layer swallows errors on purpose, so "no console errors" proves nothing. Check for real delivery from the page:

```js
// initialised? (the localStorage key only exists after a successful init)
Object.keys(localStorage).filter((k) => k.startsWith('mp_'))

// did anything actually go out? Mixpanel batches, so wait ~6s first
performance.getEntriesByType('resource')
  .map((e) => e.name)
  .filter((n) => /api-js\.mixpanel/.test(n))
```

Two traps when checking a change to the embed:

- **`cache-control: max-age=600`.** After redeploying `embed/analytics.js` the browser serves the old copy for ten minutes — including its old errors. Force a fresh fetch with `'?cb=' + Date.now()` (and `delete window.dhAnalytics` first, or the double-init guard bails).
- **Mixpanel batches.** Nothing hits the network for ~5s after a `track()`. An immediate check always shows zero.
