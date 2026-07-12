---
title: "Building a Blog with Jekyll on GitHub Pages"
date: 2026-07-11
lang: en
ref: building-jekyll
tags: [tech, jekyll, tutorial]
---

[Jekyll](https://jekyllrb.com/) is a static site generator that turns Markdown into a beautiful website. Combined with [GitHub Pages](https://pages.github.com/), it's the perfect setup for a personal blog.

## Why Jekyll?

- **No database** — everything is static files
- **Markdown support** — write posts in plain text
- **Free hosting** — GitHub Pages serves everything for free
- **Custom domains** — point your own domain at it

## How it works

Write a Markdown file, save it in `_posts/` with the right naming convention, push to GitHub, and your site updates automatically. That's it.

### Naming convention

```
_posts/YYYY-MM-DD-your-title-here.md
```

Each post starts with frontmatter:

```yaml
---
title: "Your Post Title"
date: 2026-07-11
tags: [tech, tutorial]
---
```

## Multi-language setup

This site supports English, Chinese, and Japanese. Each post declares its language in the frontmatter:

```yaml
---
title: "Hello"
lang: en
ref: hello-world
---
```

The `ref` field links translations together. Posts with the same `ref` but different `lang` values are recognized as translations of each other.

---

That's all for now. Happy writing! 🚀
