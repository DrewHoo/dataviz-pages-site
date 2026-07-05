# Register a project on the drewhoover.com index

Read this when a sibling site is live and ready to appear on `https://drewhoover.com/` — i.e. it should show up as a card on the homepage and at `/projects/<slug>/`. This touches a different repo (`DrewHoo/DrewHoo.github.io`), so it's usually done as a follow-up commit after the project itself ships.

## Add the project card

In `~/Projects/DrewHoo.github.io/`, create `src/content/projects/<slug>.md`:

```markdown
---
title: My New Thing
blurb: One-sentence description, max 280 chars. Shows up on the homepage card.
liveUrl: https://drewhoover.com/<slug>/
repoUrl: https://github.com/DrewHoo/<slug>
stack:
  - Vite
  - React
  - D3
pinned: false
order: 10            # higher sorts earlier
updated: YYYY-MM-DD
---

Optional MDX body renders at /projects/<slug>/.
```

Commit just that file (be careful — the index repo may have unrelated in-progress edits to other cards; do not include them). Push. The index rebuilds via its own Actions workflow.
