# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a **Hexo 5.x** static blog site using the **NexT 7.8.0** theme. Content is written in Chinese. Deployed to GitHub Pages via Travis CI at `blog.underpinetree.com`.

## Commands

```bash
npm install        # Install dependencies
npm run server     # Local dev server (hexo server)
npm run build      # Generate static files (hexo generate)
npm run clean      # Remove generated files (hexo clean)
npm run deploy     # Deploy to GitHub Pages (hexo deploy)
```

No project-level lint or test scripts exist.

## Architecture

- **`_config.yml`** — Hexo site configuration
- **`source/_posts/`** — Blog posts (Markdown with YAML front-matter: title, date, tags)
- **`source/assets/img/`** — Post images and avatar
- **`themes/next/`** — NexT theme (Swig templates, Stylus CSS, i18n)
- **`themes/next/_config.yml`** — Theme configuration (~997 lines; schemes, sidebar, comments, analytics)
- **`scaffolds/`** — Templates for new posts/pages/drafts

Hexo reads `source/` and `_config.yml`, renders Markdown via `hexo-renderer-marked`, applies theme templates, and writes output to `public/` (gitignored).

## Deployment

Travis CI runs `hexo generate` on push to `master` and deploys `public/` to GitHub Pages. The `CNAME` file preserves the custom domain.

## AI Workflow Artifacts

The repo contains OpenSpec commands and skills in `.claude/` and `.cursor/` directories. These are for AI-driven development workflows, not part of the blog content.
