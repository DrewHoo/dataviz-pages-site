# Lazy-loaded Mixpanel analytics

Read this when adding pageview + structured-event tracking to a sibling site. The drewhoover.com data-viz sites share a Mixpanel project; the pattern keeps the main bundle small and degrades silently if Mixpanel is blocked.

## The shared token

The client token is **public by design** (write-only event ingestion) and safe to commit. As of writing: `1c6a0f45b8a5768185a8d9a2f4d65452`.

`mixpanel-browser` is ~130KB gzipped — large enough to hurt initial paint. **Code-split it into its own chunk** via dynamic import.

## `src/analytics.js`

```js
const TOKEN = '1c6a0f45b8a5768185a8d9a2f4d65452'

let mp = null
const queue = []

function flush() {
  for (const args of queue) { try { mp.track(...args) } catch {} }
  queue.length = 0
}

if (typeof window !== 'undefined') {
  import('mixpanel-browser')
    .then(m => {
      mp = m.default
      mp.init(TOKEN, {
        // Auto-fire pageviews on initial load AND on every history API
        // URL change (including replaceState). Each shareable URL view
        // counts as its own pageview.
        track_pageview: 'url-with-path-and-query-string',
      })
      flush()
    })
    .catch(() => {
      // Adblockers commonly block scripts with 'mixpanel' in the URL.
      // Drop queued events silently — analytics failure must never break
      // the app.
      queue.length = 0
    })
}

export function track(name, props) {
  if (mp) { try { mp.track(name, props) } catch {} }
  else queue.push([name, props])
}
```

## `src/main.jsx`

```js
import './analytics.js'  // side-effect: kicks off the lazy load
```

## `src/App.jsx`

```js
import { track } from './analytics.js'

// Fire a structured event when meaningful state changes. Pageviews
// answer "what URL did they land on"; structured events make "which
// item did they look at" a one-click query in Mixpanel.
useEffect(() => {
  if (!data) return
  track('Chart viewed', {
    ticker: data.symbol,
    name: data.name,
    category: data.category,
    range: rangeCode,
    zoomed: !!zoom,
  })
}, [data])
```

## What you get

- **Auto-pageviews**: one on first load (Mixpanel captures `$referrer` automatically), one on every `replaceState` URL change.
- **Structured events**: rich properties for filtering / segmenting in the Mixpanel UI.
- **Adblock-safe**: a blocked Mixpanel chunk fails silently; the app keeps working.
- **Tiny critical bundle**: main bundle stays unchanged; the Mixpanel chunk loads after first paint.

Pairs naturally with `references/url-state.md` — the `replaceState` URL changes from that pattern automatically generate pageviews here.
