# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

The GitHub Pages user site of Saibal Roy, live at https://saibal-roy.github.io/. Plain HTML and CSS, no build step. Project sites such as `/docling-batch-extract/` are published from their own repositories; this one only links to them.

| Path | What it is |
|---|---|
| `index.html` | Home page: the story, "How I build", and one card per project |
| `github-practices/index.html` | How I publish open source on GitHub: each practice tagged Done or Next, with proof links and a short "How" |
| `prompts.md` | A pointer to the build prompts. The prompts live only in docling-batch-extract's `prompts.md`; never copy them here |
| `.github/workflows/pages.yml` | Publishes `index.html`, `.nojekyll`, `assets/` and every top-level folder with its own `index.html`, after a link crawl |
| `.github/social-preview.png` | 1280 x 640 image for Settings, General, Social preview (uploaded by hand; GitHub has no API for it) |
| `assets/og-image.png` | The same image, published with the site as the home page's `og:image` (link previews on LinkedIn, X and Slack). Replace both together |

## Commands

```bash
../textextraxt/scripts/preview_site.sh          # serve this site at localhost:8080/ with the project at /docling-batch-extract/, and crawl both
../textextraxt/scripts/preview_site.sh --stop
python3 scripts/check_site.py http://localhost:8080/ --no-search --ignore docling-batch-extract/
```

Headless Chrome screenshots can't go below 500 px wide, so check mobile layouts at 500 px.

## Rules

- **New pages** go in their own folder with an `index.html` (like `github-practices/`), reuse the colour tokens and light/dark styles from `index.html`, and link back home. Run the preview crawl before pushing.
- **The practices page must match reality.** Before changing a Done or Next tag, check the setting through the GitHub API on the repositories it covers (docling-batch-extract, saibal-roy.github.io, saibalroy-em).
- **Prompts improve themselves**, but not here: improvements go into docling-batch-extract's `prompts.md` and its improvements log.
- **Writing:** Saibal's own voice, plain and specific; no emojis and no em-dashes anywhere. Every number is measured or sourced.
- **Employer:** his company may appear only in career-history sentences (the story on the home page). Never as a label: not in titles, meta descriptions, link-preview text or images, project cards or the practices page. A label without it must not read as working at AWS ("I lead a DevOps team that runs client platforms on AWS").
- **Commits:** only when asked, authored as Saibal Roy <connectsaibalroy@gmail.com>, with no co-author or AI trailer.
