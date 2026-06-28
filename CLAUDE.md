# William Hugo Blog — Project Context

Hugo blog deployed at **https://williamlwclwc.github.io/william-hugo/**

## Deployment

Push to `main` → GitHub Actions builds → auto-deploys to GitHub Pages. No manual build step needed.

## Content Structure

```
content/
├── en/
│   ├── posts/       ← English articles
│   └── about/       ← About page (headless bundle)
└── zh/
    ├── posts/       ← Chinese articles (same slug as EN)
    └── about/       ← Chinese about page
```

Both language versions of the same post **must share the same filename slug** (e.g., `my-post.md`) so Hugo's `relLangURL` language switcher works correctly.

## Post Frontmatter

```yaml
---
title: "Post Title"
date: YYYY-MM-DD
description: "One sentence description shown in post card"
tags: ["tag1", "tag2"]
categories: ["Category"]
---
```

- `date`: today's date unless the post specifies otherwise
- `tags`: lowercase, descriptive (e.g., `["golang", "tutorial"]`)
- `categories`: title-case, broad topic (e.g., `["Technology"]` / `["技术"]`)
- Chinese posts use Chinese tags and categories

## Craft Integration

Post pipeline is managed via the **Hugo Blog Posts** collection in Craft:

- Collection ID: `7FA2F76C-8A5A-4861-8084-1BF9D2687CB7`
- Document ID: `539C0BD0-6A5D-429A-A8F2-4CBE95E2B500`

Collection schema:
| Field | Type | Notes |
|---|---|---|
| title | text | Post title (in the language it was written) |
| type | singleSelect | e.g. `Technical` / `Life` |
| tags | multiSelect | Article tags |
| status | singleSelect | `Draft` → `Published` |
| comments | text | Description or extra notes for Claude |

## Publish Workflow

Use the `/publish-post` skill. The steps are:

1. Read the Craft collection to find the requested post item and its ID
2. If the post content is in a linked Craft document, read it; otherwise use content provided by user
3. Produce **English** and **Chinese** versions (translate whichever is missing)
4. Choose a URL slug: lowercase, hyphenated, English words
5. Write `content/en/posts/{slug}.md` and `content/zh/posts/{slug}.md` with correct frontmatter
6. `git add content/ && git commit -m "post: {english title}" && git push`
7. Update the Craft item: `collections items-update --collection 7FA2F76C-8A5A-4861-8084-1BF9D2687CB7 --id {itemId} --status Published`

## Key Files

| File | Purpose |
|---|---|
| `hugo.toml` | Site config, language settings, nav items |
| `data/socials.toml` | Social links — shared by About page and article footers |
| `layouts/partials/nav.html` | Nav override: home links + About/关于 i18n |
| `layouts/partials/footerRight.html` | Footer: language switcher + dark/light toggle |
| `layouts/partials/footerLeft.html` | Footer: copyright |
| `layouts/partials/share.html` | Article footer social icons (reads socials.toml) |
