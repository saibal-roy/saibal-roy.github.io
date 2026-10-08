# saibal-roy.github.io

The GitHub Pages **user site** for [Saibal Roy](https://www.saibalroy.com/): a static profile page served at **https://saibal-roy.github.io/**. It introduces how I build open source: from real production requirements to reusable business solutions designed for cost-effectiveness and reliability. It links to each project's documentation, starting with [docling-batch-extract](https://saibal-roy.github.io/docling-batch-extract/).

| File | Purpose |
|------|---------|
| `index.html` | The page: plain HTML/CSS, no build step, light and dark mode, mobile-friendly |
| `github-practices/index.html` | How I publish open source on GitHub: the checklist, with each practice linked to where it's implemented in docling-batch-extract. Update the Done and Next tags when a setting changes |
| `prompts.md` | How I build: the principles, and links to the prompts. The prompts themselves live only in docling-batch-extract's `prompts.md`, so there is one copy to improve |
| `.nojekyll` | Tells GitHub Pages to serve the files as they are (no Jekyll processing) |
| `.github/workflows/pages.yml` | On every push to `main`: stages the published files, checks every link, then deploys |
| `scripts/check_site.py` | The link checker (same as the project's). Links into project sites (`docling-batch-extract/`) are skipped, because those deploy from their own repositories |

## Preview locally

From the `docling-batch-extract` project folder **next to this one** (both in the same parent folder):

```bash
cd ../docling-batch-extract      # whatever that project folder is called locally
scripts/preview_site.sh          # finds this folder at ../saibal-roy.github.io (or set USER_SITE=/path)
```

It serves this folder at `http://localhost:8080/` and the project site at `http://localhost:8080/docling-batch-extract/`, the same layout as GitHub Pages, and checks every link.

## Publish

1. Create a **public** repository named exactly `saibal-roy.github.io`.
2. Settings → Pages → Build and deployment → Source: **GitHub Actions** (one time).
3. Push this folder to `main`. The *Pages* workflow checks the links and deploys. It runs again on **every push to `main`** (or by hand: Actions → Pages → Run workflow).
4. It goes live at https://saibal-roy.github.io/ within a minute or two.

Don't set a custom domain here unless you mean to. A custom domain on the user site also moves every project site (for example to `https://<domain>/docling-batch-extract/`).

To add a project: copy its `<article class="card">` block in `index.html` and change the name, description and links. To add a page: create a folder with its own `index.html` (like `github-practices/`); the Pages workflow publishes every top-level folder that has one.
