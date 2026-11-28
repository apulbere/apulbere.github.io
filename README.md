# Personal blog [adrian.md](https://adrian.md)

Built with [Hugo](https://gohugo.io) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme (git submodule in `themes/PaperMod`).

```sh
git clone --recurse-submodules git@github.com:apulbere/apulbere.github.io.git
hugo server -M       # preview at http://localhost:1313 (in memory; drafts: -D)
```

New posts go in `content/posts/YYYY-MM-DD-slug.md`; the date and URL slug come from the filename
(`/YYYY/MM/DD/slug/`), so don't rename published posts. Minimal front matter:

```yaml
---
title: My Post
description: One-sentence summary for search results and social cards (optional, falls back to the opening paragraph).
tags: [java, spring]
ShowToc: true # optional: show a table of contents for this post
---
```

Pushing to `master` builds and deploys via `.github/workflows/hugo.yml`.
