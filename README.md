# theinit01.github.io — Source for my blog

This repository is the source for my personal blog (the site published at `theinit01.github.io`). It contains the Hugo site sources, a theme, templates, and the generated site output.

Purpose

- Host and publish personal posts, project write-ups, and a static showcase.
- Keep content and site templates together so the site can be built locally or in CI.

High-level architecture

- `content/` — Markdown content for posts, pages, and section indexes (this is where most writing happens).
- `archetypes/` — default front matter templates used when creating new content.
- `layouts/` — site-specific template overrides and partials.
- `themes/terminal` — the theme used by this site (theme files may be vendorized here).
- `static/` — static assets copied to the site root (images, icons, extra files).
- `public/` — generated static site output (what gets uploaded to the web server). This folder is created by `hugo` and should not be edited directly.
- `hugo.toml` — site configuration (format may be TOML; check the file for params like baseURL, theme, and menus).

How the site works (brief)

- Hugo reads Markdown files from `content/`, applies templates from `layouts/` and the active theme, and writes static HTML/CSS/JS into `public/`.
- Templates and partials in `layouts/` override those in `themes/terminal` when present — this is the recommended way to customize site appearance.

Minimal local run & build notes

- To preview the site with live reload (recommended while writing):

```bash
hugo server -D
# open http://localhost:1313
```

- To build the production site into `public/`:

```bash
hugo --minify
```

- To serve the contents of `public/` locally (optional):

```bash
cd public && python3 -m http.server 8080
# open http://localhost:8080
```

Where to edit content and templates

- Add or edit posts under `content/posts/` (or other sections under `content/`).
- Adjust templates or partials in `layouts/` to change markup or override theme behavior.
- Update theme files in `themes/terminal` if you need to change the theme itself.
- Check `archetypes/default.md` for the default front matter fields created for new posts.

CI workflow (GitHub Actions)

- Location: `.github/workflows/hugo.yaml` — used to build and publish the site (GitHub Pages).
- Triggers: runs on pushes to `main` and can be started manually via the Actions tab (`workflow_dispatch`).
- Permissions: the workflow grants the `pages: write` and `id-token: write` permissions via the built-in `GITHUB_TOKEN` so it can configure and deploy Pages.
- Concurrency: configured to avoid overlapping deploys (group `pages`) and not cancel in-progress runs.

Summary of what the workflow does:

1. Build job
	- Installs a pinned Hugo extended CLI (the workflow sets `HUGO_VERSION` — in this repo it's 0.139.2).
	- Installs Dart Sass (used by some themes/assets).
	- Checks out the repository (with submodules) and installs Node deps if a lockfile exists.
	- Runs `hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"` to generate the site into `public/`.
	- Uploads the generated `public/` directory as an artifact for deployment.

2. Deploy job
	- Runs after the build job and uses `actions/deploy-pages` to publish the artifact to GitHub Pages (the job sets the `github-pages` environment URL from the deployment output).

How to trigger and inspect

- Push to `main` to trigger a normal deploy, or start the workflow manually from the Actions tab.
- Check the Actions run logs in your GitHub repo to see install, build, and deploy steps and any failures.

Notes & references

- Hugo documentation: https://gohugo.io/
- Theme folder: `themes/terminal`
- Config: `hugo.toml`





