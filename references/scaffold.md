# Scaffold a new sibling site

Read this when setting up a brand-new repo under `<owner>/<slug>` and getting it serving at `<domain>/<slug>/`. Walks the path from `gh repo create` to the first green Pages deploy.

## 1. Create the repo

```bash
gh repo create <owner>/<slug> --public --clone
cd <slug>
```

Use a slug that will read well in the URL. Prefer lowercase, kebab-case.

## 2. Bootstrap with Vite (recommended)

```bash
npm create vite@latest . -- --template react   # plain JS to match cfb-records / buy-it-now-or-never
npm install
```

JSX (not TS) matches the other sibling projects' conventions; TS is fine if there's a real reason. Other stacks (Astro, plain HTML, Observable Framework) work too — the rest of this skill still applies as long as you end up with a static `dist/` folder.

## 3. Set the Vite `base` path — do not skip

Edit `vite.config.js`:

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  base: '/<slug>/',   // matches the repo name and the URL path
  plugins: [react()],
})
```

Without this, the built `index.html` references `/assets/foo.js` (absolute from domain root) and blows up because the site is actually served from `/<slug>/assets/foo.js`. Symptom: blank page with 404s for every JS/CSS file in the browser console.

If you reference static assets directly from JS source, use `import.meta.env.BASE_URL` to build URLs (e.g. `fetch(\`${import.meta.env.BASE_URL}data/index.json\`)`) so Vite rewrites them at build time.

## 4. Add the deploy workflow

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to GitHub Pages

on:
  push:
    branches: ['main']
  workflow_dispatch:
  # Optional: scheduled data refresh. Omit if there's no per-day data.
  schedule:
    - cron: '30 22 * * 1-5'   # weekdays 22:30 UTC, after US market close

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: 'pages'
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'
      - run: npm ci
      - run: npm run build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: ./dist

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

The `schedule` trigger is the cheapest way to refresh time-sensitive data — see `references/build-time-data.md`.

**Gotcha — lightningcss / sharp on Linux CI.** Projects using Tailwind v4 (or anything pulling in `lightningcss`) and projects using `sharp` ship platform-specific binaries as optional deps. When the lockfile was generated on macOS and `npm ci` runs on a Linux runner, the Linux variant sometimes gets skipped and the build dies with `Cannot find module '../lightningcss.linux-x64-gnu.node'` (or similar for sharp). Fix by installing the binary explicitly between `npm ci` and `npm run build`:

```yaml
      - run: npm ci
      - run: npm install --no-save lightningcss-linux-x64-gnu
      - run: npm run build
```

Same template for `@rollup/rollup-linux-x64-gnu`, `@swc/core-linux-x64-gnu`, etc. Note: as of mid-2026, `actions/setup-node@v4` will warn about Node 20 deprecation — ignore for now; upgrading to a v5 action is its own task.

## 5. Enable Pages + set source to Actions

```bash
gh api -X POST repos/<owner>/<slug>/pages -f build_type=workflow 2>&1 || \
  gh api -X PUT  repos/<owner>/<slug>/pages -f build_type=workflow
```

(Pages has to be created the first time with `POST`; existing Pages config uses `PUT`. The OR handles either case.)

Or in the UI: **Settings → Pages → Build and deployment → Source: GitHub Actions**.

## 6. Push and verify

```bash
git push origin main
gh run watch --repo <owner>/<slug>
# After green:
curl -sI --resolve <domain>:443:185.199.108.153 https://<domain>/<slug>/ | head -5
# Expect HTTP/2 200, server: GitHub.com
```

Once the site responds 200, move on to `references/meta-and-assets.md` for the head/favicon/OG pass, and `references/index-registration.md` to add it to the index site's project list.
