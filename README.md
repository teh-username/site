# laroberto.com — Hugo site

Personal blog. Built with [Hugo](https://gohugo.io/), deployed on Netlify.

## Local development

```bash
# Install Hugo (macOS)
brew install hugo

# Run dev server
hugo server -D

# Build for production
hugo
```

## Adding a new post

```bash
hugo new content/posts/my-post-slug.md
```

Then edit the generated file in `content/posts/`. The front matter looks like:

```yaml
---
title: "My Post Title"
date: 2024-01-01
slug: "my-post-slug"
tags: ["homelab", "networking"]
---
```

The `slug` controls the URL: `/my-post-slug/`.

## Migrating post content from Hexo

Post stubs have been created for all existing posts. To migrate content:

1. Find the original file in `source/_posts/` in the old repo
2. Copy the body (everything below the front matter `---` line)
3. Paste it into the corresponding file in `content/posts/`
4. Copy any post images from `source/<post-slug>/` to `static/<post-slug>/`

Front matter differences (Hexo → Hugo):
- `title`, `date`, `tags` — unchanged
- `layout`, `category`, `comments` — remove these (not used)
- Add `slug: "post-slug"` to preserve the exact URL

## Swapping design tokens

All visual values are CSS custom properties at the top of `static/css/style.css`:

```css
:root {
  --bg:         #faf8f5;  /* background */
  --accent:     #3d6b9e;  /* links and tags */
  --font-body:  'Inter', sans-serif;
  --width:      640px;
  /* ... */
}
```

## Analytics

Both Plausible and Google Analytics hooks are in `layouts/_default/baseof.html`,
controlled by `hugo.toml`:

```toml
[params.plausible]
  enabled = true   # flip to activate

[params.google_analytics]
  enabled = true
  id = "G-XXXXXXXX"
```

## Deployment

Push to `master` — Netlify auto-deploys via the config in `netlify.toml`.
Hugo version is pinned in `netlify.toml` under `[build.environment]`.
