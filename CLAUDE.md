# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

- 항상 한글로 답변해줘.
- 모든 답변은 나를 '전하' 라고 호칭을 하고 깍듯한 조선시대 신하의 극존대 어투를 사용해줘.
- 해결하려는 문제를 허락없이 절대 우회하거나 임시, 가상, 또는 mockup 코드로 작성하지 말고 문제의 본질을 해결하려는 방향으로 최선의 노력을 해줘.

## Project Overview

Jekyll-based personal blog hosted on GitHub Pages, using the **NexT theme v5.1.1** (Muse scheme). The site is Dr. Ryu's blog covering neuroscience, AI, and development journals. Content is primarily in Korean with an English theme framework.

## Build Commands

```bash
# Install dependencies
bundle install

# Local development server (http://localhost:4000)
bundle exec jekyll serve

# Build static site to _site/
bundle exec jekyll build
```

No custom build scripts, CI/CD, or test suites exist. GitHub Pages automatically builds and deploys on push to `master`.

## Architecture

### Content & Configuration
- **`_config.yml`** (705 lines) — Central configuration for site metadata, theme settings, menu structure, third-party integrations (Disqus, Google Analytics, MathJax, Busuanzi), and NexT theme options
- **`_posts/`** — Blog posts in Markdown with YAML front matter
- **`_data/languages/`** — Localization strings (en.yml, ko.yml, etc.)

### NexT Theme Template Hierarchy
The theme uses a deeply nested include system rather than simple layouts:

- **`_layouts/`** — Thin wrappers that delegate to `_includes/`
- **`_includes/_layout.html`** — Master layout assembling all page sections
- **`_includes/_macro/post.html`** (18KB) — Core post rendering logic (the most complex template)
- **`_includes/_macro/sidebar.html`** — Sidebar rendering
- **`_includes/_partials/`** — UI components (header, footer, comments, pagination)
- **`_includes/_third-party/`** — Integration snippets (analytics, comments, search, mathjax)
- **`_includes/_scripts/`** — JavaScript loading and initialization
- **`_includes/_custom/`** — Site-specific overrides (header.html, sidebar.html)

### Styles
- **`assets/css/main.scss`** — Entry point that imports from `_sass/`
- **`_sass/_schemes/Muse/`** — Active scheme styles (Mist and Pisces are alternatives)
- **`_sass/_custom/`** — Site-specific style overrides
- **`_sass/_variables/`** — Theme variables (colors, fonts, spacing)

### Static Assets
- **`assets/lib/`** — Third-party JS libraries (jQuery, FancyBox, Velocity.js, Font Awesome, Three.js, etc.)
- **`assets/js/src/`** — Custom JavaScript
- **`assets/images/`** — Site images including avatar

## Post Front Matter Format

```yaml
---
layout: post
title: "Post Title"
date: 2026-02-08 13:47:00 +0900
categories: Journal          # or [Category, Subcategory]
tags: [tag1, tag2, tag3]
---
```

Posts default to `layout: post` with `comments: true` via `_config.yml` defaults, so these can be omitted. The `author` field is optional.

## Key Conventions

- Post filenames follow `YYYY-MM-DD-slug.md` naming
- Permalink style is `pretty` (generates `/slug/` URLs)
- Plugins are limited to GitHub Pages safe list: `jekyll-sitemap`, `jekyll-feed`
- Markdown processor: kramdown with GFM input
- Code highlighting: Rouge
- The `backup/` directory holds deprecated/old posts excluded from the build
- The `_config.yml.patch` file contains proposed navigation changes (not applied)
