# dataviz-pages-site

A [Claude Code](https://claude.com/claude-code) skill. Build, scaffold, or troubleshoot a
data-viz project that deploys to GitHub Pages under your own domain at `<domain>/<repo-slug>/`.
Covers the full stack: repo + Vite base-path config, Actions deploy and scheduled data refreshes,
build-time data fetching, agent-researched datasets with per-row citations, shareable URL state,
mobile-first responsive design, SEO (prerendering so crawlers see content), OG/Twitter meta tags +
favicon + preview image generation, analytics via a shared embed, common date-handling traps, and
registering the project card on your index site.

See [`SKILL.md`](SKILL.md) and the deep-dives in [`references/`](references/).

## What it assumes

- A GitHub User Site repo (`<owner>/<owner>.github.io`) with a custom domain, built from the
  companion starter, `DrewHoo/personal-site-starter`. Every other public repo you own with Pages
  enabled then serves at `<domain>/<repo>/` with no extra DNS.
- The index site serves the shared embeds (`/embed/back-bar.js`, `/embed/giscus.js`,
  `/embed/analytics.js`) that project sites load with one script tag each.

## Install

```sh
git clone git@github.com:DrewHoo/dataviz-pages-site.git ~/.claude/skills/dataviz-pages-site
```

Or, if you keep skills as repos with the `skills` script from `DrewHoo/skills-manager`:

```sh
skills clone git@github.com:DrewHoo/dataviz-pages-site.git
```

## Configure

Edit the **Config** table at the top of `SKILL.md`: your GitHub username, your domain, and the
local path of your index repo clone. Everything else in the skill refers to those three values as
`<owner>`, `<domain>`, and `<index_repo_path>`. The example repos listed under "Canonical examples"
are real public repos and stay as they are.

## Contributing

Fixes flow both ways. If a troubleshooting entry saved you, or a step was wrong, open a PR.
