# Publish Hugo Blog Post from Craft

Publish one or more posts from the Craft "Hugo Blog Posts" collection to the Hugo site.

## Trigger

User says something like:
- "publish the post about X"
- "/publish-post"
- "把 Craft 里的 X 文章发布出去"

## Steps

### 1. Find the post in Craft

```
collections items-get --collection 7FA2F76C-8A5A-4861-8084-1BF9D2687CB7 --format markdown
```

Match the item by title to the one the user mentioned. Note its `id`, `title`, `tags`, `type`, and `comments`.

If the content is not in the collection item, check if the user provided a Craft document link. If so:
```
documents resolve-link <craft-url>
blocks get <rootBlockId> --format markdown
```

Otherwise ask the user for the article body.

### 2. Prepare both language versions

- If written in **English**: translate to Chinese
- If written in **Chinese**: translate to English
- If both are provided: use as-is

Each version needs: title, description (1 sentence), body, tags, categories.
- EN: tags/categories in English, title-case categories
- ZH: tags/categories in Chinese, same semantic meaning

### 3. Choose a slug

- Lowercase, hyphenated, English words
- Based on the English title (e.g., "My First Post" → `my-first-post`)
- Keep it short (3–5 words max)

### 4. Write the Hugo files

**`content/en/posts/{slug}.md`**
```markdown
---
title: "{English title}"
date: {YYYY-MM-DD}
description: "{English description}"
tags: ["{tag1}", "{tag2}"]
categories: ["{Category}"]
---

{English body}
```

**`content/zh/posts/{slug}.md`**
```markdown
---
title: "{中文标题}"
date: {YYYY-MM-DD}
description: "{中文描述}"
tags: ["{标签1}", "{标签2}"]
categories: ["{分类}"]
---

{中文正文}
```

### 5. Commit and push

```bash
git add content/en/posts/{slug}.md content/zh/posts/{slug}.md
git commit -m "post: {english title}"
git push
```

### 6. Update Craft status

```
collections items-update --collection 7FA2F76C-8A5A-4861-8084-1BF9D2687CB7 --id {itemId} --status Published
```

### 7. Confirm

Tell the user:
- Live URL (after ~1 min GitHub Actions): `https://williamlwclwc.github.io/william-hugo/posts/{slug}/`
- Chinese URL: `https://williamlwclwc.github.io/william-hugo/zh/posts/{slug}/`
- Craft status updated to Published
