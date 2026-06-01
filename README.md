# ryanlail.github.io

Personal blog of Ryan Lail, built with [Jekyll](https://jekyllrb.com/) and
served by [GitHub Pages](https://pages.github.com/).

## Writing a post

Add a Markdown file to `_posts/` named `YYYY-MM-DD-title.md` with front matter:

```yaml
---
layout: post
title: "Your title"
date: 2026-05-28
tags: [research]
---
```

## Running locally

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://localhost:4000>.

GitHub Pages builds the `main` branch automatically — no Actions workflow
required, since the site uses only Pages-supported plugins.
