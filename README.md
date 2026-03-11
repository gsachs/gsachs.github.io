# Building With AI — Blog

Personal blog built on GitHub Pages + Jekyll. No frameworks, no build tools, no dependencies beyond what GitHub Pages provides natively.

## Setup

1. Create a new repo on GitHub named `yourusername.github.io`
2. Push this directory to that repo
3. Go to **Settings → Pages → Source** and select `main` branch
4. Wait ~60 seconds for the first build
5. Visit `https://yourusername.github.io`

## Writing a new post

Create a file in `_posts/` with the naming format:

```
_posts/YYYY-MM-DD-your-post-slug.md
```

Front matter template:

```yaml
---
layout: post
title: "Your Title Here"
subtitle: "Optional subtitle shown on index and below title"
date: YYYY-MM-DD
series: "Building With AI"
---

Your content here in markdown.
```

The `series` field is optional. If present, posts in the same series show a linked list at the bottom of each article.

## Structure

```
_config.yml          # Site settings
_layouts/
  default.html       # Base layout (head, nav, footer)
  post.html          # Article layout (title, date, content, series)
_posts/              # Articles (markdown)
assets/css/
  style.css          # All styles, no preprocessor
index.html           # Homepage listing all posts
```

## Customization

- **Site title/description:** edit `_config.yml`
- **Fonts:** swap Google Fonts imports in `_layouts/default.html`
- **Colors:** CSS variables at the top of `assets/css/style.css`
- **Add an about page:** create `about.md` with `layout: default` front matter
