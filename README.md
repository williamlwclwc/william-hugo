# William Liu's Blog

Personal bilingual blog (English / 中文) built with Hugo and deployed to GitHub Pages.

**Live site:** https://williamlwclwc.github.io/william-hugo/

---

## Tech Stack

| Layer | Tool |
|---|---|
| Static site generator | [Hugo](https://gohugo.io/) |
| Theme | [Dream](https://github.com/g1eny0ung/hugo-theme-dream) |
| CI/CD | GitHub Actions |
| Hosting | GitHub Pages |
| Post management | Craft (Hugo Blog Posts collection) |

---

## Writing a New Post

Every post lives in two files with the same filename slug:

```
content/en/posts/my-post-slug.md   ← English version
content/zh/posts/my-post-slug.md   ← Chinese version
```

### Frontmatter

```yaml
---
title: "Your Post Title"
date: 2026-06-28
description: "A one-sentence summary shown in the post card."
tags: ["tag1", "tag2"]
categories: ["Category"]
---

Your content here...
```

### Publishing

1. Add a new row to the **Hugo Blog Posts** collection in Craft with the post title, type, and tags
2. Tell Claude: *"publish the Craft post titled 'X'"*
3. Claude will translate, write both files, commit, push, and mark the post as Published in Craft
4. GitHub Actions deploys automatically — the post is live in ~1 minute

---

## Project Structure

```
william-hugo/
├── content/
│   ├── en/
│   │   ├── posts/          English articles
│   │   └── about/          About page content (English)
│   └── zh/
│       ├── posts/          Chinese articles
│       └── about/          About page content (Chinese)
├── data/
│   └── socials.toml        Social links (About page + article footer)
├── layouts/
│   └── partials/           Theme overrides (nav, footer, share buttons)
├── static/
│   └── img/
│       └── avatar.png      Profile avatar
├── themes/
│   └── dream/              Hugo theme (git submodule)
├── .github/
│   └── workflows/
│       └── hugo.yml        GitHub Actions deploy workflow
└── hugo.toml               Site configuration
```

---

## Configuration

### Adding / editing social links

Edit `data/socials.toml`. Icons use [Ionicons](https://ionic.io/ionicons) names.

```toml
socials = [
  { href = "https://yoursite.com", target = "_blank", icon = "home-outline", title = "Homepage" },
  { href = "https://github.com/you", target = "_blank", icon = "logo-github", title = "GitHub" },
]
```

Changes appear in the About page social card and at the bottom of every article.

### Changing the colour theme

Edit `lightTheme` / `darkTheme` in `hugo.toml`. Any [DaisyUI theme](https://daisyui.com/docs/themes/) name works.

```toml
[params]
lightTheme = "emerald"
darkTheme  = "forest"
```

### Updating About page content

- English: `content/en/about/me.md` (add more `.md` files for extra cards)
- Chinese: `content/zh/about/me.md`

---

## Local Development

```bash
# Install Hugo (macOS)
brew install hugo

# Clone with theme submodule
git clone --recurse-submodules https://github.com/williamlwclwc/william-hugo.git
cd william-hugo

# Serve locally with live reload
hugo server -D
```

Open http://localhost:1313/william-hugo/
