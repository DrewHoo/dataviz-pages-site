# Shareable URL state + share button

Read this when adding shareable views to an interactive site — every meaningful UI choice (selected item, filter, zoom range) should round-trip through the URL so users can copy/share the current view. Includes the paired "Share this chart" button pattern.

## Reading on mount

```js
function readInitialState() {
  if (typeof window === 'undefined') return { /* defaults */ }
  const params = new URLSearchParams(window.location.search)
  const selected = params.get('t') || null
  const range = RANGE_CODES[params.get('r') || ''] ?? null
  let zoom = null
  const z = params.get('z')
  if (z && z.includes(',')) {
    const [a, b] = z.split(',')
    const d1 = new Date(`${a}T00:00:00Z`)
    const d2 = new Date(`${b}T00:00:00Z`)
    if (!isNaN(d1) && !isNaN(d2) && d1 < d2) zoom = [d1, d2]
  }
  return { selected, range, zoom }
}

export default function App() {
  const initial = useMemo(() => readInitialState(), [])
  const [selected, setSelected] = useState(initial.selected)
  // ...
}
```

`useMemo(() => readInitialState(), [])` runs once on mount before paint; useState initializer alternative works too.

## Validating against loaded data

URL params are untrusted. Validate after the catalog loads:

```js
useEffect(() => {
  if (!index) return
  const known = new Set(index.items.map(t => t.id))
  if (!selected || !known.has(selected)) setSelected(DEFAULT_ID)
}, [index, selected])
```

## Mirroring state → URL

```js
useEffect(() => {
  if (!selected) return
  const params = new URLSearchParams()
  if (selected !== DEFAULT_ID) params.set('t', selected)
  if (rangeCode !== 'all') params.set('r', rangeCode)
  if (zoom) params.set('z', `${iso(zoom[0])},${iso(zoom[1])}`)
  const qs = params.toString()
  const next = qs ? `${location.pathname}?${qs}` : location.pathname
  if (next !== location.pathname + location.search) {
    history.replaceState(null, '', next)
  }
}, [selected, rangeCode, zoom])
```

**Use `replaceState`, not `pushState`.** Every interaction should NOT add a back-button entry — clicking 20 tickers shouldn't require 20 back presses to return. The URL stays current and shareable; history stays clean.

**Omit defaults.** If `selected === DEFAULT_ID`, leave `t` out of the URL. Keeps shared URLs short and the homepage URL clean.

## User-action wrappers vs URL-load preservation

A common subtle issue: clicking a different ticker should clear the zoom (those date ranges don't carry over), but loading a URL with both `t=` and `z=` should preserve the zoom. Solution: route UI events through small wrappers that clear dependent state, but let URL-read set the initial state directly.

```js
function selectTicker(sym) {
  if (sym === selected) return
  setZoom(null)    // user-initiated selection clears the zoom
  setSelected(sym)
}
function changeRange(years) {
  setZoom(null)    // user-initiated range change clears the zoom
  setRange(years)
}
// Initial mount: setSelected(initial.selected) directly — no clearing.
```

## Share button

Pair URL state with an obvious "Share this chart" button. Don't make users hunt the address bar.

```jsx
function ShareButton() {
  const [copied, setCopied] = useState(false)
  function share() {
    const url = window.location.href
    track('Share clicked', { url })  // see references/analytics.md
    const done = () => { setCopied(true); setTimeout(() => setCopied(false), 1800) }
    if (navigator.clipboard?.writeText) {
      navigator.clipboard.writeText(url).then(done, done)
    } else {
      // Fallback for old / non-HTTPS contexts.
      const ta = document.createElement('textarea')
      ta.value = url
      document.body.appendChild(ta)
      ta.select()
      try { document.execCommand('copy') } catch {}
      document.body.removeChild(ta)
      done()
    }
  }
  return (
    <button onClick={share} className={copied ? 'share share--copied' : 'share'}>
      {copied ? '✓ Link copied' : 'Share this chart'}
    </button>
  )
}
```

Place it in the controls bar next to the other view-modifying controls (range pills, filters), not orphaned in a corner.
