---
name: dataviz-pages-site
description: Build, scaffold, or troubleshoot a sibling data-viz project that deploys to GitHub Pages under drewhoover.com/<repo-slug>/. Covers the whole stack — repo + Vite + base-path config, Actions-based deploy and scheduled data refreshes, build-time data fetching, shareable URL state, mobile-first responsive design, OG/Twitter meta tags + favicon + preview image generation, lazy-loaded Mixpanel analytics, common date-handling traps, and registering the project on the drewhoover.com index. Use when the user says "new dataviz project", "set up a new site under drewhoover.com", "make a dataviz site", "why isn't my project serving", "make this site shareable", "add OG image to a dataviz site", "add analytics to a dataviz site", "fix mobile UX on a dataviz site", or mentions the space-rock / cfb-all-time-records / buy-it-now-or-never pattern.
---

# Building a GitHub Pages data-viz project site under drewhoover.com

The index site lives at `DrewHoo/DrewHoo.github.io` and serves `https://drewhoover.com/`. Any other repo owned by `DrewHoo` with GitHub Pages enabled automatically appears at `https://drewhoover.com/<repo-name>/` — no DNS per project, no subdomain. This skill describes how to make a new sibling site, and how to diagnose a broken one.

Canonical examples in the wild:
- `DrewHoo/space-rock` → https://drewhoover.com/space-rock/ (Vite + React + DuckDB-WASM)
- `DrewHoo/cfb-all-time-records` → https://drewhoover.com/cfb-all-time-records/ (Vite + React, multi-page)
- `DrewHoo/buy-it-now-or-never` → https://drewhoover.com/buy-it-now-or-never/ (Vite + React + D3, scheduled Yahoo fetch, URL-state, Mixpanel — the most complete reference)

When in doubt about a craft-level concern (analytics, OG image, URL state, mobile, etc.), copy the pattern from `buy-it-now-or-never`.

## The rules the routing depends on

1. **Repo name = URL path.** `DrewHoo/my-thing` serves at `drewhoover.com/my-thing/`. Pick the slug you want in the URL.
2. **Must be public** (or a Pro org with Pages). User Site routing only fans out to public repos under the same owner.
3. **Build output must match the base path.** See `references/scaffold.md` (step 3) — getting this wrong is the #1 reason a new site ships broken asset URLs.

## Reference files

Each file under `references/` covers one concern in depth. Read the ones relevant to the current task; don't preload them all.

- **`references/scaffold.md`** — Read when setting up a new repo. Covers `gh repo create`, Vite + `base` path, the deploy workflow YAML (with the lightningcss/sharp Linux CI gotcha), enabling Pages via API, push + verify.
- **`references/meta-and-assets.md`** — Read when polishing a site for sharing. Full `<head>` template (title, description, OG, Twitter, favicons, drewhoover.com chrome) plus `sharp`-based favicon and OG image generation scripts.
- **`references/index-registration.md`** — Read when a site is live and needs a project card on `drewhoover.com`. Touches a different repo (`DrewHoo.github.io`).
- **`references/build-time-data.md`** — Read when the site has time-varying JSON content (prices, sports stats, scraped data). Covers the fetch-and-bake-at-build-time pattern, bounded concurrency, scheduled refresh cron tuning, and why `public/data/` is gitignored.
- **`references/url-state.md`** — Read when adding shareable views. URL ↔ state mirroring with `replaceState`, validation against loaded data, user-action vs URL-load preservation, and the paired "Share this chart" button.
- **`references/mobile-and-dates.md`** — Read when interactions break on touch devices or dates display one day off for US viewers. Covers `pointer*` events, `touch-action: pan-y`, responsive table hiding, and the UTC midnight pitfall (`scaleUtc` + `utcFormat`).
- **`references/analytics.md`** — Read when adding Mixpanel. Lazy-loaded chunk, shared public token, auto-pageviews on `replaceState`, structured events, adblock-safe fallback.

## Scaffold workflow (high-level)

For a brand-new site, walk these steps in order. Each step is a one-liner here; the deep reference is in `references/scaffold.md` unless noted.

1. `gh repo create DrewHoo/<slug> --public --clone && cd <slug>`
2. `npm create vite@latest . -- --template react && npm install`
3. **Set `base: '/<slug>/'` in `vite.config.js`.** Skipping this ships a blank page with 404s on every asset.
4. Add `.github/workflows/deploy.yml` (Actions → Pages). If using `lightningcss` / `sharp` / Tailwind v4, add the Linux binary fix between `npm ci` and `npm run build`.
5. Enable Pages: `gh api -X POST repos/DrewHoo/<slug>/pages -f build_type=workflow` (fall back to `PUT` if it already exists).
6. Fill out `index.html` from the template in `references/meta-and-assets.md` (drewhoover.com back-bar + giscus chrome go in `<head>`).
7. Generate favicon set + OG image via `sharp` scripts. See `references/meta-and-assets.md`.
8. Register the project on the drewhoover.com index. See `references/index-registration.md`.
9. `git push origin main && gh run watch --repo DrewHoo/<slug>`, then `curl -sI https://drewhoover.com/<slug>/` to confirm 200.

Add the opt-in subsystems (build-time data, URL state, mobile interactions, analytics) as the project's needs justify — see the matching reference file.

## Style baseline

The drewhoover.com sites have a shared visual feel but no design system. Hand-rolled CSS in `src/styles.css`, system font stack, neutral palette plus one or two accent colors that come from the project's content (e.g. buy-it-now-or-never's red/green semantic palette).

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

## Pre-launch checklist

Before sharing the URL anywhere:

- [ ] Site loads at `https://drewhoover.com/<slug>/` (200, not 404)
- [ ] No console errors on load
- [ ] Mobile: hover-style interactions work on touch (scrub with a finger, not just a mouse) — see `references/mobile-and-dates.md`
- [ ] Mobile: page is readable without horizontal scroll
- [ ] Selecting a different item / filter updates the URL (look at the address bar) — see `references/url-state.md`
- [ ] Reloading the URL with state lands you in the same view
- [ ] Share button copies the current URL and shows feedback
- [ ] Pasting the site URL into a chat/social platform shows the OG card with the right image, title, and description (use Facebook Sharing Debugger / Twitter Card Validator before announcing) — see `references/meta-and-assets.md`
- [ ] Favicon is visible in the tab strip (hard refresh if stale)
- [ ] Back-bar appears at the top of the page
- [ ] Comments section renders at the bottom (giscus widget loads)
- [ ] Project card is registered on drewhoover.com index and links work — see `references/index-registration.md`

## Troubleshooting

### Blank page, 404s for every asset in the browser console
You forgot `base: '/<slug>/'` in `vite.config.js` — or the base string doesn't match the repo name exactly. Fix, rebuild, redeploy. See `references/scaffold.md` step 3.

### Back-bar / comments don't appear
1. **Scripts present in the deployed HTML?** `curl -s --resolve drewhoover.com:443:185.199.108.153 https://drewhoover.com/<slug>/ | grep drewhoover.com/embed` — if empty, redeploy.
2. **Old `gh-pages` branch deploy still active?** If `package.json` has a `deploy:pages` script that copies `dist → docs/`, the live site is the stale `docs/`. Migrate to the Actions workflow, remove `docs/`.
3. **Client-side exception on load?** The embed scripts are `async` in `<head>`; an early app error keeps them from visibly mounting. Check the browser console.

### Deploy workflow runs green but the site didn't actually update
```bash
gh api repos/DrewHoo/<slug>/pages --jq '{source, build_type}'
```
If `build_type: "legacy"`, Pages is still serving from a branch. Fix:
```bash
gh api -X PUT repos/DrewHoo/<slug>/pages -f build_type=workflow
```
Then re-run the workflow.

### OG image cached at old version after redesign
Reddit / X / Bluesky / LinkedIn cache OG fetches for hours-days. Force a refresh via the platform's card validator (Facebook Sharing Debugger, Twitter Card Validator) before announcing, or share with a cache-busting query param the first time. See `references/meta-and-assets.md`.

### Hover works on desktop but does nothing on phone
You're using `onMouseMove` / `onMouseDown`. Switch to `onPointer*` and branch on `pointerType` — see `references/mobile-and-dates.md`.

### Dates display one day earlier than they should
UTC midnight pitfall — see `references/mobile-and-dates.md`. Use `scaleUtc` + `utcFormat` from d3.

### "NotServedByPagesError" banner in Settings → Pages after a DNS change
UI caches the last health-check. Hard refresh the page; if persistent, remove and re-add the custom domain in the Pages UI to force a re-check. The index repo's `public/CNAME` will re-seed it on the next deploy.

## What not to bother with

- Per-repo custom domain — each project inherits `drewhoover.com/<slug>/` for free.
- Per-repo `CNAME` file — only the index repo needs one.
- Installing giscus app on every project repo — the embed points all traffic at the central Discussions instance on `DrewHoo.github.io`.
- Building a full design system — the sibling sites share a feel through convention, not a shared CSS package.
- Adding consent / cookie banners for Mixpanel — the token is anonymous-by-default, no PII; the sibling sites do not currently require consent in any jurisdiction we operate in.
- Per-project favicon CDN — favicons are static files in `public/`.
- Server-side rendering / Next.js — overkill. Vite static is enough for the data-viz scope.
