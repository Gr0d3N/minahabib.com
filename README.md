# minahabib.com

Personal site and blog, built with [Hugo](https://gohugo.io/) and the
[PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

## Writing a post

Add a Markdown file under `content/posts/`:

```
content/posts/my-post.md
```

With front matter:

```yaml
---
title: "My Post Title"
date: 2026-11-15
draft: false
tags: ["topic"]
---
```

Set `draft: true` to keep a post out of the published site.

## Local preview

```bash
hugo server -D
```

Opens at http://localhost:1313/.

## Deploy

Pushing to `master` triggers the GitHub Actions workflow
(`.github/workflows/deploy.yml`), which builds the site and deploys it to
GitHub Pages at https://www.minahabib.com/.

## Theme updates

The theme is a git submodule. To update it:

```bash
git submodule update --remote --merge
```
