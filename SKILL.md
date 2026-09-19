---
name: dataviz-pages-site
description: Build, scaffold, or troubleshoot a sibling data-viz project that deploys to GitHub Pages under your own domain at <domain>/<repo-slug>/. Covers the whole stack — repo + Vite + base-path config, Actions-based deploy and scheduled data refreshes, build-time data fetching, agent-researched datasets with per-row citations, shareable URL state, mobile-first responsive design, SEO (prerendering the SPA so crawlers see content, headings, titles, JSON-LD, sitemap and robots), OG/Twitter meta tags + favicon + index-card image generation, Mixpanel analytics via the index site's shared embed, common date-handling traps, and registering or refreshing the project card on the index site. Use when the user says "new dataviz project", "set up a new site under my domain", "make a dataviz site", "why isn't my project serving", "make this site shareable", "add OG image to a dataviz site", "add analytics to a dataviz site", "I'm not getting any analytics/Mixpanel events", "fix SEO on a dataviz site", "my project isn't showing up in search", "update the project card on the index site", "fix mobile UX on a dataviz site", says the data "does not exist anywhere" or has to be researched/assembled from documents, or mentions the space-rock / cfb-all-time-records / buy-it-now-or-never / how-many-rings pattern.
---

# Building a GitHub Pages data-viz project site under your own domain

The index site lives at `<owner>/<owner>.github.io` and serves `https://<domain>/`. Any other repo owned by `<owner>` with GitHub Pages enabled automatically appears at `https://<domain>/<repo-name>/` — no DNS per project, no subdomain. This skill describes how to make a new sibling site, and how to diagnose a broken one.

## Config

Set these once for your setup. Every `<owner>`, `<domain>`, and `<index_repo_path>` in this skill and its references means these values. `<slug>` stays per-project.

| key | value | meaning |
| --- | --- | --- |
| `owner` | `DrewHoo` | GitHub username. The index site repo is always `<owner>/<owner>.github.io`. |
| `domain` | `drewhoover.com` | Custom domain on the index site. Project sites serve at `https://<domain>/<slug>/`. |
| `index_repo_path` | `~/Projects/DrewHoo.github.io` | Local clone of the index repo, for registering project cards. |

The Mixpanel token is not config here. It lives in the index repo's embed source (`src/scripts/embed-analytics.js`), and project sites only load the built embed. See `references/analytics.md`.

Canonical examples in the wild:
- `DrewHoo/hostile-territory` → https://drewhoover.com/hostile-territory/ — the reference for a **computed-and-verified** dataset (bulk sources joined against researched tenures, every displayed claim receipt-checked against an independent source) and for the current dense/quiet visual direction.
- `DrewHoo/space-rock` → https://drewhoover.com/space-rock/ (Vite + React + DuckDB-WASM)
- `DrewHoo/cfb-all-time-records` → https://drewhoover.com/cfb-all-time-records/ (Vite + React, multi-page)
- `DrewHoo/buy-it-now-or-never` → https://drewhoover.com/buy-it-now-or-never/ (Vite + React + D3, scheduled Yahoo fetch, URL-state, Mixpanel — the most complete reference)
- `DrewHoo/how-many-rings` → https://drewhoover.com/how-many-rings/ — the reference for a **researched** dataset: no source file existed, so an agent fleet assembled ~1,300 cited rows from media guides and archived staff directories. See `references/researched-datasets.md`.

When in doubt about a craft-level concern (analytics, OG image, URL state, mobile, etc.), copy the pattern from `buy-it-now-or-never`.

## The rules the routing depends on

1. **Repo name = URL path.** `<owner>/my-thing` serves at `<domain>/my-thing/`. Pick the slug you want in the URL.
2. **Must be public** (or a Pro org with Pages). User Site routing only fans out to public repos under the same owner.
3. **Build output must match the base path.** See `references/scaffold.md` (step 3) — getting this wrong is the #1 reason a new site ships broken asset URLs.

## Reference files

Each file under `references/` covers one concern in depth. Read the ones relevant to the current task; don't preload them all.

- **`references/scaffold.md`** — Read when setting up a new repo. Covers `gh repo create`, Vite + `base` path, the deploy workflow YAML (with the lightningcss/sharp Linux CI gotcha), enabling Pages via API, push + verify.
- **`references/meta-and-assets.md`** — Read when polishing a site for sharing. Full `<head>` template (title, description, OG, Twitter, favicons, index-site chrome) plus `sharp`-based favicon and OG image generation scripts.
- **`references/index-registration.md`** — Read when a site is live and needs a project card on `<domain>`. Touches a different repo (`<owner>.github.io`).
- **`references/build-time-data.md`** — Read when the site has time-varying JSON content (prices, sports stats, scraped data). Covers the fetch-and-bake-at-build-time pattern, bounded concurrency, scheduled refresh cron tuning, and why `public/data/` is gitignored.
- **`references/researched-datasets.md`** — Read when the data does **not** exist as a file or API anywhere and has to be assembled by an agent fleet from documents. Covers inverting the question so a deterministic join does the counting, per-row verbatim citations as a hallucination gate, encoding judgment as versioned rule files, the coverage report that catches the failure modes a finished page hides, and how to prompt sweeps so they reject as well as confirm.
- **`references/url-state.md`** — Read when adding shareable views. URL ↔ state mirroring with `replaceState`, validation against loaded data, user-action vs URL-load preservation, and the paired "Share this chart" button.
- **`references/mobile-and-dates.md`** — Read when interactions break on touch devices or dates display one day off for US viewers. Covers `pointer*` events, `touch-action: pan-y`, responsive table hiding, and the UTC midnight pitfall (`scaleUtc` + `utcFormat`).
- **`references/analytics.md`** — Read when adding Mixpanel, **or when a site is reporting nothing**. Project sites do not inherit the index site's analytics; several shipped without any and reported zero for months. Covers the shared `/embed/analytics.js` drop-in, the per-repo alternative, and how to verify delivery instead of assuming it.
- **`references/cfb-sources.md`** — Read for any college-football project: which bulk sources curl cleanly, cfbfastR's specific lies (UTC dates, 2001-2007 neutral flags), the no-key ESPN adjudicator APIs, Sports-Reference via the Wayback Machine, and the one-color logo pipeline.
- **`references/seo.md`** — Read before announcing a site, or when it isn't showing up in search. Prerendering the SPA (the big one — without it the deployed HTML contains no content at all), a real `<h1>`, titles aimed at winnable queries, JSON-LD, canonical, sitemap and robots.

## Scaffold workflow (high-level)

For a brand-new site, walk these steps in order. Each step is a one-liner here; the deep reference is in `references/scaffold.md` unless noted.

1. `gh repo create <owner>/<slug> --public --clone --template DrewHoo/dataviz-project-template && cd <slug>`. The template ships Vite + React, the deploy workflow, the prerender step, the `<head>` generator, the favicon and OG scripts, and a sample D3 chart with URL state. `references/scaffold.md` explains what each piece does if you need to build one by hand.
2. Set `"name"` in `package.json` to `<slug>`. `vite.config.js` derives the `base` path from it, and a wrong name ships a blank page with 404s on every asset.
3. Edit `site.config.js`: `<domain>`, title, description, OG copy, accent color.
4. `npm install && npm run gen:favicon && npm run gen:og`. Open `public/og.png` and `public/card.png` and look at them. Commit the outputs.
5. Enable Pages: `gh api -X POST repos/<owner>/<slug>/pages -f build_type=workflow` (fall back to `PUT` if it already exists).
6. If using `lightningcss` / `sharp` in the build / Tailwind v4, uncomment the Linux binary line in `.github/workflows/deploy.yml`.
7. Replace the sample: `src/data/sample.js`, `src/App.jsx`, `src/Chart.jsx`, `src/styles.css`. Keep `hydrateRoot`, keep `window` reads out of render, keep pointer events.
8. Register the project on the index site. See `references/index-registration.md`.
9. `git push origin main && gh run watch --repo <owner>/<slug>`, then `curl -sI https://<domain>/<slug>/` to confirm 200.

Add the opt-in subsystems (build-time data, URL state, mobile interactions, analytics) as the project's needs justify — see the matching reference file. Analytics is close to non-optional: it is one script tag, and skipping it is why several shipped sites have no data at all.

## Style baseline

The sibling sites have a shared visual feel but no design system. Hand-rolled CSS in `src/styles.css`, system font stack, neutral palette plus one or two accent colors that come from the project's content (e.g. buy-it-now-or-never's red/green semantic palette).

A working baseline:

```css
:root {
  --fg: #1a1a1a;
  --fg-muted: #555;
  --bg: #fafafa;
  --bg-card: #ffffff;
  --border: #e6e6e6;
  --accent: #1f4e8c;
}

* { box-sizing: border-box; }

html, body {
  margin: 0; padding: 0;
  background: var(--bg); color: var(--fg);
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', system-ui, sans-serif;
  font-size: 15px; line-height: 1.5;
}

main { max-width: 1080px; margin: 0 auto; padding: 40px 24px 80px; }
```

Card-style sections (white background, 1px border, 8px radius, light shadow) read well against the off-white page background and don't compete with the data. Use real semantic colors (red for danger, green for success, etc.) for data elements, not for chrome.

### The denser direction (hostile-territory, 2026-09)

Drew's standing preference moved toward **less text, less color, less chrome**
— the data is the page. What that meant concretely, and what to reach for
first on the next project:

- Strike framing prose on sight. Drew cut the hero stat, the origin-story
  paragraph, a decorative header label, and a self-narrating interaction
  caption from an already-short page. One italic sentence of setup, then the
  board.
- Identity through assets, not labels: entity marks (team logos as one-color
  stamps) replaced both the per-game W/L letters and the spelled-out school
  names next to coaches. Outcome rides on LIGHTNESS (lit cream chip vs
  recessive charcoal), not hue — colorblind-safe by construction, and the
  page stays duotone-plus-one-accent.
- Records as `3–7 .300` baseball averages on one line, not "30% won" on two.
- Poster-dense layout: two columns flowing down-then-across, 18px marks, 6px
  row padding. This started as a static PNG export Drew loved so much the
  site was rebuilt to match it — make the dense export early; it's a design
  probe, not just a share asset.
- A `scripts/gen-poster.mjs`-style dense PNG export earns its keep three
  ways: Reddit share asset, the `og:image` (keep `twitter:image` on the 2:1
  card — X force-crops), and the design target itself. Bake the site URL and
  date into its footer so attribution survives rehosting.
- Interactions follow the same restraint: a popover carries score/coaches/OT
  detail so the board itself never grows labels; hovering a game highlights
  the same host's other games with a ring, silently.
- Mockups-first still works: three themed looks as static HTML, screenshot,
  let Drew pick, then iterate the one encoding he flags. Validate data colors
  with the dataviz skill's palette validator against the chosen surface
  before showing anything.

## When the data has to be researched

If the dataset does not exist as a file, an API, or a scrapable table — if the
answer is scattered across documents and has to be assembled — read
`references/researched-datasets.md` before starting. The short version: invert
the question so a small fixed list joins against researched rows, make every row
carry the verbatim line that proves it, keep judgment in versioned rule files
rather than in agent heads, and ship a coverage report, because a page with a
third of its rows missing looks exactly as confident as a complete one.

## Pre-launch checklist

Before sharing the URL anywhere:

- [ ] Site loads at `https://<domain>/<slug>/` (200, not 404)
- [ ] No console errors on load
- [ ] **Content is in the HTML, not just in JS**: `curl -s <url> | grep -c '<div id="root"></div>'` returns `0`. A `1` means crawlers see an empty page — see `references/seo.md`
- [ ] Exactly one `<h1>`, and it names the subject rather than a section
- [ ] `<title>` targets a query this page can plausibly win, not the head term owned by Wikipedia
- [ ] `<link rel="canonical">` present
- [ ] Analytics actually delivering — `Object.keys(localStorage).filter(k => k.startsWith('mp_'))` is non-empty and a `track()` produces a request to `api-js.mixpanel.com` (wait ~6s; it batches). See `references/analytics.md`
- [ ] Mobile: hover-style interactions work on touch (scrub with a finger, not just a mouse) — see `references/mobile-and-dates.md`
- [ ] Mobile: page is readable without horizontal scroll
- [ ] Selecting a different item / filter updates the URL (look at the address bar) — see `references/url-state.md`
- [ ] Reloading the URL with state lands you in the same view
- [ ] If the page has a share button: it copies the current URL and shows feedback (a mostly-static page can reasonably skip the button and still keep the URL state)
- [ ] Pasting the site URL into a chat/social platform shows the OG card with the right image, title, and description (use Facebook Sharing Debugger / Twitter Card Validator before announcing) — see `references/meta-and-assets.md`
- [ ] Favicon is visible in the tab strip (hard refresh if stale)
- [ ] Back-bar appears at the top of the page
- [ ] Comments section renders at the bottom (giscus widget loads)
- [ ] Project card is registered on the index site and links work — see `references/index-registration.md`
- [ ] Card cover is 1200×750, and **open the PNG** — the card crops 8:5, so a 1200×630 OG image loses its top and bottom silently
- [ ] Card blurb describes the site as it is now, not as it was before the last redesign
- [ ] If the data was researched rather than downloaded: coverage report is clean, every displayed claim links to a source, and any weaker-evidence rows are labeled as such — see `references/researched-datasets.md`
- [ ] If the page shows photos of people: no image is a site logo or the wrong person (check aspect ratios; landscape "headshots" are usually an `og:image` fallback)

## Troubleshooting

### Blank page, 404s for every asset in the browser console
You forgot `base: '/<slug>/'` in `vite.config.js` — or the base string doesn't match the repo name exactly. Fix, rebuild, redeploy. See `references/scaffold.md` step 3.

### Back-bar / comments don't appear
1. **Scripts present in the deployed HTML?** `curl -s --resolve <domain>:443:185.199.108.153 https://<domain>/<slug>/ | grep <domain>/embed` — if empty, redeploy.
2. **Old `gh-pages` branch deploy still active?** If `package.json` has a `deploy:pages` script that copies `dist → docs/`, the live site is the stale `docs/`. Migrate to the Actions workflow, remove `docs/`.
3. **Client-side exception on load?** The embed scripts are `async` in `<head>`; an early app error keeps them from visibly mounting. Check the browser console.

### Deploy workflow runs green but the site didn't actually update
```bash
gh api repos/<owner>/<slug>/pages --jq '{source, build_type}'
```
If `build_type: "legacy"`, Pages is still serving from a branch. Fix:
```bash
gh api -X PUT repos/<owner>/<slug>/pages -f build_type=workflow
```
Then re-run the workflow.

### OG image cached at old version after redesign
Reddit / X / Bluesky / LinkedIn cache OG fetches for hours-days. Force a refresh via the platform's card validator (Facebook Sharing Debugger, Twitter Card Validator) before announcing, or share with a cache-busting query param the first time. See `references/meta-and-assets.md`.

### A project site sends no Mixpanel events
It almost certainly never had analytics. Project sites are separate Pages deployments and inherit nothing from the index site; `grep -c "embed/analytics.js" index.html` returning `0` is the answer. See `references/analytics.md`.

### The site isn't in Google at all
Check `curl -s <url> | grep -c '<div id="root"></div>'`. If it returns `1`, the deployed HTML has no content and there is nothing to index. See `references/seo.md`.

### Hover works on desktop but does nothing on phone
You're using `onMouseMove` / `onMouseDown`. Switch to `onPointer*` and branch on `pointerType` — see `references/mobile-and-dates.md`.

### Dates display one day earlier than they should
UTC midnight pitfall — see `references/mobile-and-dates.md`. Use `scaleUtc` + `utcFormat` from d3.

### "NotServedByPagesError" banner in Settings → Pages after a DNS change
UI caches the last health-check. Hard refresh the page; if persistent, remove and re-add the custom domain in the Pages UI to force a re-check. The index repo's `public/CNAME` will re-seed it on the next deploy.

## What not to bother with

- Per-repo custom domain — each project inherits `<domain>/<slug>/` for free.
- Per-repo `CNAME` file — only the index repo needs one.
- Installing giscus app on every project repo — the embed points all traffic at the central Discussions instance on `<owner>.github.io`.
- Building a full design system — the sibling sites share a feel through convention, not a shared CSS package.
- Adding consent / cookie banners for Mixpanel — the token is anonymous-by-default, no PII; the sibling sites do not currently require consent in any jurisdiction we operate in.
- Per-project favicon CDN — favicons are static files in `public/`.
- Server-side rendering / Next.js — overkill. Vite static is enough for the data-viz scope.
