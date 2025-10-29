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