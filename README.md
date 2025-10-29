# plebeiusgaragicus.github.io

A personal blog built with [MkDocs](https://www.mkdocs.org/) and [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

## Local Development

1. Install dependencies:
```bash
pip install -r requirements.txt
```

2. Run the development server:
```bash
mkdocs serve
```

3. Open your browser to `http://127.0.0.1:8000`

## Writing Blog Posts

Create new blog posts in `docs/blog/posts/` with the following frontmatter:

```markdown
---
date: YYYY-MM-DD
authors:
  - plebeiusgaragicus
categories:
  - Category Name
tags:
  - tag1
  - tag2
---

# Post Title

Your content here...

<!-- more -->

Extended content after the break...
```

## Adding Static HTML Pages

MkDocs automatically copies all non-markdown files from `docs/` to the built site. To add standalone HTML tools or pages:

1. **Create the HTML file** in `docs/` (e.g., `docs/bitcoin-price.html`)
2. **Create an index post** to list your tools (see `docs/blog/posts/plebtools.md`)
3. **Link to static pages** using absolute paths: `/bitcoin-price.html`

**Pattern**: Use a blog post (like "PlebTools") as a curated index/sitemap for static utilities. This keeps them discoverable while maintaining the blog's content flow.

## Deployment

The site automatically deploys to GitHub Pages when you push to the main branch via GitHub Actions.

## Project Structure

```
.
├── docs/
│   ├── index.md          # Home page
│   ├── about.md          # About page
│   ├── tags.md           # Tags page
│   └── blog/
│       ├── index.md      # Blog index
│       ├── .authors.yml  # Author information
│       └── posts/        # Blog posts go here
├── mkdocs.yml            # MkDocs configuration
└── requirements.txt      # Python dependencies
```