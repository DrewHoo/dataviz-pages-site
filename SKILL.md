---
name: dataviz-pages-site
description: Build, scaffold, or troubleshoot a sibling data-viz project that deploys to GitHub Pages under your own domain at <domain>/<repo-slug>/. Covers the drewhoover.com plumbing — repo + Vite + base-path config, Actions-based deploy and scheduled data refreshes, OG/Twitter meta tags + favicon + index-card image generation, Mixpanel analytics via the index site's shared embed, registering or refreshing the project card on the index site, and the pre-launch checklist. Use when the user says "new dataviz project", "set up a new site under my domain", "make a dataviz site", "why isn't my project serving", "add OG image to a dataviz site", "add analytics to a dataviz site", "I'm not getting any analytics/Mixpanel events", "my project isn't showing up in search", "update the project card on the index site", or mentions the space-rock / cfb-all-time-records / buy-it-now-or-never / hostile-territory pattern. Craft concerns (mobile/touch, URL state, SPA prerender SEO, build-time data, the dense style direction) live in the dataviz-site-craft skill; datasets that must be researched or row-verified live in the receipts-research skill — this skill is the hosting and chrome around both.
---

# Building a GitHub Pages data-viz project site under your own domain

The index site lives at `<owner>/<owner>.github.io` and serves `https://<domain>/`. Any other repo owned by `<owner>` with GitHub Pages enabled automatically appears at `https://<domain>/<repo-name>/` — no DNS per project, no subdomain. This skill describes how to make a new sibling site, and how to diagnose a broken one.

Two sibling skills carry the host-agnostic halves; load them alongside this one as the task needs:

- **`dataviz-site-craft`** — mobile/touch interactions, the UTC date pitfall, shareable URL state + share button, SPA prerendering and SEO, build-time data fetching, the style baseline and the dense/quiet direction.
- **`receipts-research`** — datasets that don't exist anywhere and must be assembled by an agent fleet with per-row receipts, and row-by-row verification of computed datasets (coverage reports, corrections layers, fleet prompting).

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
- `DrewHoo/how-many-rings` → https://drewhoover.com/how-many-rings/ — the reference for a **researched** dataset: no source file existed, so an agent fleet assembled ~1,300 cited rows from media guides and archived staff directories. See the receipts-research skill.

When in doubt about a craft-level concern, copy the pattern from `buy-it-now-or-never`.

## The rules the routing depends on

1. **Repo name = URL path.** `<owner>/my-thing` serves at `<domain>/my-thing/`. Pick the slug you want in the URL.
2. **Must be public** (or a Pro org with Pages). User Site routing only fans out to public repos under the same owner.
3. **Build output must match the base path.** See `references/scaffold.md` (step 3) — getting this wrong is the #1 reason a new site ships broken asset URLs.

## Reference files

Each file under `references/` covers one concern in depth. Read the ones relevant to the current task; don't preload them all.

- **`references/scaffold.md`** — Read when setting up a new repo. Covers `gh repo create`, Vite + `base` path, the deploy workflow YAML (with the lightningcss/sharp Linux CI gotcha), enabling Pages via API, push + verify.
- **`references/meta-and-assets.md`** — Read when polishing a site for sharing. Full `<head>` template (title, description, OG, Twitter, favicons, index-site chrome) plus `sharp`-based favicon and OG image generation scripts.
- **`references/index-registration.md`** — Read when a site is live and needs a project card on `<domain>`. Touches a different repo (`<owner>.github.io`).
- **`references/analytics.md`** — Read when adding Mixpanel, **or when a site is reporting nothing**. Project sites do not inherit the index site's analytics; several shipped without any and reported zero for months. Covers the shared `/embed/analytics.js` drop-in, the per-repo alternative, and how to verify delivery instead of assuming it.
- **`references/cfb-sources.md`** — Read for any college-football project: which bulk sources curl cleanly, cfbfastR's specific lies (UTC dates, 2001-2007 neutral flags), the no-key ESPN adjudicator APIs, Sports-Reference via the Wayback Machine, and the one-color logo pipeline.

Concerns that used to live here and moved: URL state, mobile/dates, SPA prerender SEO, and build-time data are the dataviz-site-craft skill's references; researched datasets are the receipts-research skill.

## Scaffold workflow (high-level)

For a brand-new site, walk these steps in order. Each step is a one-liner here; the deep reference is in `references/scaffold.md` unless noted.

1. `gh repo create <owner>/<slug> --public --clone --template DrewHoo/dataviz-project-template && cd <slug>`. The template ships Vite + React, the deploy workflow, the prerender step, the `<head>` generator, the favicon and OG scripts, and a sample D3 chart with URL state. `references/scaffold.md` explains what each piece does if you need to build one by hand.
2. Set `"name"` in `package.json` to `<slug>`. `vite.config.js` derives the `base` path from it, and a wrong name ships a blank page with 404s on every asset.
3. Edit `site.config.js`: `<domain>`, title, description, OG copy, accent color.
4. `npm install && npm run gen:favicon && npm run gen:og`. Open `public/og.png` and `public/card.png` and look at them. Commit the outputs.
5. Enable Pages: `gh api -X POST repos/<owner>/<slug>/pages -f build_type=workflow` (fall back to `PUT` if it already exists).
6. If using `lightningcss` / `sharp` in the build / Tailwind v4, uncomment the Linux binary line in `.github/workflows/deploy.yml`.
7. Replace the sample: `src/data/sample.js`, `src/App.jsx`, `src/Chart.jsx`, `src/styles.css`. Keep `hydrateRoot`, keep `window` reads out of render, keep pointer events (the dataviz-site-craft skill covers why).
8. Register the project on the index site. See `references/index-registration.md`.
9. `git push origin main && gh run watch --repo <owner>/<slug>`, then `curl -sI https://<domain>/<slug>/` to confirm 200.

Add the opt-in subsystems (build-time data, URL state, mobile interactions — dataviz-site-craft; analytics — `references/analytics.md`) as the project's needs justify. Analytics is close to non-optional: it is one script tag, and skipping it is why several shipped sites have no data at all.

## Pre-launch checklist

Before sharing the URL anywhere:

- [ ] Site loads at `https://<domain>/<slug>/` (200, not 404)
- [ ] No console errors on load
- [ ] **Content is in the HTML, not just in JS**: `curl -s <url> | grep -c '<div id="root"></div>'` returns `0`. A `1` means crawlers see an empty page — see dataviz-site-craft's seo reference
- [ ] Exactly one `<h1>`, and it names the subject rather than a section
- [ ] `<title>` targets a query this page can plausibly win, not the head term owned by Wikipedia
- [ ] `<link rel="canonical">` present
- [ ] Analytics actually delivering — `Object.keys(localStorage).filter(k => k.startsWith('mp_'))` is non-empty and a `track()` produces a request to `api-js.mixpanel.com` (wait ~6s; it batches). See `references/analytics.md`
- [ ] Mobile: hover-style interactions work on touch (scrub with a finger, not just a mouse) — see dataviz-site-craft
- [ ] Mobile: page is readable without horizontal scroll
- [ ] Selecting a different item / filter updates the URL (look at the address bar) — see dataviz-site-craft's url-state reference
- [ ] Reloading the URL with state lands you in the same view
- [ ] If the page has a share button: it copies the current URL and shows feedback (a mostly-static page can reasonably skip the button and still keep the URL state)
- [ ] Pasting the site URL into a chat/social platform shows the OG card with the right image, title, and description (use Facebook Sharing Debugger / Twitter Card Validator before announcing) — see `references/meta-and-assets.md`
- [ ] Favicon is visible in the tab strip (hard refresh if stale)
- [ ] Back-bar appears at the top of the page
- [ ] Comments section renders at the bottom (giscus widget loads)
- [ ] Project card is registered on the index site and links work — see `references/index-registration.md`
- [ ] Card cover is 1200×750, and **open the PNG** — the card crops 8:5, so a 1200×630 OG image loses its top and bottom silently
- [ ] Card blurb describes the site as it is now, not as it was before the last redesign
- [ ] If the data was researched rather than downloaded: coverage report is clean, every displayed claim links to a source, and any weaker-evidence rows are labeled as such — see the receipts-research skill
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
Check `curl -s <url> | grep -c '<div id="root"></div>'`. If it returns `1`, the deployed HTML has no content and there is nothing to index. See dataviz-site-craft's seo reference.

### Hover works on desktop but does nothing on phone
You're using `onMouseMove` / `onMouseDown`. Switch to `onPointer*` and branch on `pointerType` — see dataviz-site-craft's mobile-and-dates reference.

### Dates display one day earlier than they should
UTC midnight pitfall — see dataviz-site-craft's mobile-and-dates reference. Use `scaleUtc` + `utcFormat` from d3.

### "NotServedByPagesError" banner in Settings → Pages after a DNS change
UI caches the last health-check. Hard refresh the page; if persistent, remove and re-add the custom domain in the Pages UI to force a re-check. The index repo's `public/CNAME` will re-seed it on the next deploy.

## What not to bother with

- Per-repo custom domain — each project inherits `<domain>/<slug>/` for free.
- Per-repo `CNAME` file — only the index repo needs one.
- Installing giscus app on every project repo — the embed points all traffic at the central Discussions instance on `<owner>.github.io`.
- Building a full design system — the sibling sites share a feel through convention, not a shared CSS package (dataviz-site-craft's style reference is the baseline).
- Adding consent / cookie banners for Mixpanel — the token is anonymous-by-default, no PII; the sibling sites do not currently require consent in any jurisdiction we operate in.
- Per-project favicon CDN — favicons are static files in `public/`.
- Server-side rendering / Next.js — overkill. Vite static is enough for the data-viz scope.
