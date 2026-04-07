# Copilot Instructions for mindbyte.nl

## Project Overview

This is a Jekyll blog (mindbyte.nl) using the **Minimal Mistakes** theme with the "dark" skin. It is hosted on GitHub Pages from the `mivano/mindbyte.nl` repository.

## Build & Serve

```bash
bundle install          # install dependencies (first time)
bundle exec jekyll serve  # local dev server at http://localhost:4000
bundle exec jekyll build  # production build (output in _site/)
```

Note: `_config.yml` changes require a server restart — Jekyll does not hot-reload the config.

## Writing Posts

Posts live in `_posts/` with filename format `YYYY-MM-DD-title-slug.md`. Required frontmatter:

```yaml
---
published: 2025-06-02T12:34:24.141Z  # ISO 8601 datetime
title: "Post Title"
tags:
  - tag-name
header:
  teaser: https://mindbyte.nl/images/image-name.png
---
```

- `published` uses full ISO 8601 timestamps (not just dates)
- `header.teaser` should be an absolute URL pointing to `/images/`
- Drafts go in `_drafts/`
- Images are stored in the `images/` directory at the repo root (not `assets/images/`)

## Post Defaults (from _config.yml)

Posts automatically get these values unless overridden:

- `layout: single`, `author_profile: true`, `read_time: true`
- `comments: true`, `share: true`, `related: true`, `classes: wide`

## Permalink Structure

Posts use: `/:categories/:year/:month/:day/:title:output_ext`

Categories in frontmatter directly affect the URL path, so changing them is a breaking change for existing posts.

## Theme Customization

The Minimal Mistakes theme is vendored directly (not as a gem). Customizations:

- **Layouts**: `_layouts/` — `single.html` is the primary post layout
- **Includes**: `_includes/` — partials for comments, author profile, SEO, etc.
- **Styles**: `_sass/` — SCSS overrides; `_variables.scss` contains theme variables
- **Navigation**: `_data/navigation.yml` defines the main site navigation

## Comments

Comments use **Giscus** (GitHub Discussions-based). Configuration is in `_config.yml` under `comments.giscus`. Comments only render when `jekyll.environment == 'production'`.

## Content Authoring with Front Matter CMS

The repo includes `frontmatter.json` for the [Front Matter VS Code extension](https://frontmatter.codes/), which provides a CMS-like editing experience for posts.

## Key Conventions

- Post filenames use kebab-case with the date prefix
- Tag names are lowercase and hyphenated (e.g., `github-actions`, `azure`)
- The site uses Lunr for client-side search with full content indexing enabled
- Timezone is `Europe/Amsterdam`
