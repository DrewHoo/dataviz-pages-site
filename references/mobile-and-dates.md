# Mobile-first responsive design + the UTC midnight date pitfall

Read this when interactions work on a desktop dev machine but break in the browser environments your viewers actually use — touch devices or non-UTC timezones. Both are bugs that don't show up in local dev and embarrass the site after launch.

## Mobile-first responsive design

Phone is a first-class viewer. Most data-viz sites get shared on Reddit/X/Bluesky and clicked from a phone first.

### Viewport meta tag

In `<head>`:
```html
<meta name="viewport" content="width=device-width, initial-scale=1.0" />
```

Already present in the head template (`references/meta-and-assets.md`). Without this the page renders at 980px scaled down — broken.

### Pointer events, not mouse events

If your visualization has hover interactions, **use `pointer*` events**, not `mouse*`. `onMouseMove` does not fire on touch devices, so scrubbing a finger across a chart does nothing.

```jsx
function isTouch(e) {
  return e.pointerType === 'touch' || e.pointerType === 'pen'
}
function onPointerDown(e) {
  if (isTouch(e)) {
    // Finger down = start scrubbing the tooltip
    updateHoverAt(svgXFromEvent(e))
  } else {
    // Mouse down = start a brush-to-zoom drag
    setBrush({ startX: svgXFromEvent(e), endX: svgXFromEvent(e) })
  }
  e.currentTarget.setPointerCapture?.(e.pointerId)
}
```

`setPointerCapture` keeps events flowing to the element even when the finger slides outside its bounds — important on small screens.

### touch-action: pan-y

On the chart container CSS:
```css
.chart svg { touch-action: pan-y; }
```

Vertical page scrolls still pass through, but horizontal finger drags get captured by your scrub handler. Without this, the browser will fight your drag handler trying to scroll the page.

### Hide expensive columns / sections at small widths

Tag columns/cells with a class, then drop them on mobile:

```jsx
<th className="col-hide-mobile">Name</th>
<td className="col-hide-mobile">{t.name}</td>
```

```css
@media (max-width: 600px) {
  .col-hide-mobile { display: none; }
  .comparison th, .comparison td { padding: 5px 6px; font-size: 12px; }
}
```

Rule of thumb: keep the column you'd sort by. Drop wordy text columns (names, descriptions). Numeric columns generally fit.

### Don't pin a fixed width on tooltips

Use `width: 170px` (or whatever the wider variant needs) on the tooltip card so it stays the same shape regardless of which type of marker is hovered. Inconsistent widths flicker the layout while scrubbing — jarring on mobile.

## The UTC midnight pitfall (date handling)

If your JSON stores dates as bare `'YYYY-MM-DD'` strings (recommended — short, sortable, unambiguous), `new Date(d)` parses them as **UTC midnight**. If you then format / scale them with `timeFormat` / `scaleTime` from d3, the formatter reads in the viewer's **local timezone** — and viewers west of GMT see every date shifted back one day.

Symptom: latest data point displays as "yesterday" for US viewers. Hover labels show one day earlier than expected. ATH markers appear on the wrong date.

Fix: use the UTC-aware variants throughout:

```js
import { scaleUtc } from 'd3-scale'        // not scaleTime
import { utcFormat } from 'd3-time-format' // not timeFormat

const fmtDate = utcFormat('%b %e, %Y')
const xScale = scaleUtc().domain([...]).range([...])
```

The Date objects don't change; only how you read them does.

For non-d3 date displays, prefer `toLocaleDateString('en-US', { ..., timeZone: 'UTC' })`:

```js
function formatSince(iso) {
  const d = new Date(`${iso}T00:00:00Z`)
  return d.toLocaleDateString('en-US', {
    year: 'numeric', month: 'short', timeZone: 'UTC',
  })
}
```
