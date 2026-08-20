# Register a project on the drewhoover.com index

Read this when a sibling site is live and ready to appear on `https://drewhoover.com/` — i.e. it should show up as a card on the homepage and at `/projects/<slug>/`. This touches a different repo (`DrewHoo/DrewHoo.github.io`), so it's usually done as a follow-up commit after the project itself ships.

## Add the project card

In `~/Projects/DrewHoo.github.io/`, create `src/content/projects/<slug>.md`:

```markdown
---
title: My New Thing
blurb: "Description shown on the homepage card. Max 500 chars; *single asterisks* render as <em>."
liveUrl: https://drewhoover.com/<slug>/
repoUrl: https://github.com/DrewHoo/<slug>
tags:
  - data viz          # must be in PROJECT_TAGS (src/consts.ts) — schema rejects anything else
stack:
  - Vite
  - React
  - D3
cover: /projects/<slug>.png     # 1200×750; see meta-and-assets.md — NOT the 1200×630 og.png
coverAlt: What the cover actually shows
pinned: false
order: 10            # higher sorts earlier
created: YYYY-MM-DD  # repo creation date — the default sort key
updated: YYYY-MM-DD
---

Optional MDX body renders at /projects/<slug>/.
```

The real schema is `src/content.config.ts`; check it rather than trusting this snippet, since it's the thing that will reject the build. `tags` is a closed enum — `music` is not a tag.

Commit just that file (be careful — the index repo may have unrelated in-progress edits to other cards; do not include them). Push. The index rebuilds via its own Actions workflow.

## Keep the card in sync with the site

The card is a link from an indexed homepage, which is the best authority signal a project gets, and it's the first description a human reads. A stale one is a real cost. **When a project is redesigned, update its card in the same pass** — blurb, `coverAlt`, `cover` image, `updated`. A card still describing swimlanes and stream-sized bubbles for a page that is now a plain list is worse than no card.

Good source material for the blurb: the project's own intro copy. Adapt the author's phrasing rather than inventing marketing prose.

## Sitemap and robots live here too

Both are root-level concerns, so they belong to this repo, not the project repos.

- **`public/robots.txt`** — `@astrojs/sitemap` emits `sitemap-index.xml` + `sitemap-0.xml`, and **there is no `/sitemap.xml`**; probing that path and concluding the sitemap is missing is an easy wrong turn. robots.txt is what points crawlers at the real one:

  ```
  User-agent: *
  Allow: /

  Sitemap: https://drewhoover.com/sitemap-index.xml
  ```

- **The project sites are not in the sitemap by default.** They're separate Pages deployments, so Astro builds `/projects/<slug>/` (the card detail page) but never the actual app. `astro.config.mjs` harvests `liveUrl` from the project cards and passes them through:

  ```js
  sitemap({ customPages: siblingSites })
  ```

  A sitemap may only list URLs on its own host, so any card still pointing at `drewhoo.github.io/<slug>/` gets normalised to `drewhoover.com/<slug>/` first. Verify each URL returns 200 before including it — a sitemap full of 404s is worse than a short one.

## Analytics is not inherited either

Adding the card does not make the project report to Mixpanel. See `references/analytics.md` — the project's own `index.html` needs the `/embed/analytics.js` tag.
