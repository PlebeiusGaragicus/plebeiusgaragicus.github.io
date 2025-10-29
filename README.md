# plebeiusgaragicus.github.io

A personal site built with [MkDocs](https://www.mkdocs.org/).

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

## Adding Pages

Create markdown files in `docs/` and add them to the `nav` section in `mkdocs.yml`.

## Adding Static HTML Files

Place HTML files in `docs/` - they'll be copied as-is to the built site.

Example: `docs/tool.html` → accessible at `/tool.html`

## Deployment

Pushes to `main` automatically deploy to GitHub Pages via GitHub Actions.

## Project Structure

```
.
├── docs/
│   ├── index.md          # Home page
│   ├── about.md          # About page
│   └── *.html            # Static HTML files
├── mkdocs.yml            # MkDocs configuration
└── requirements.txt      # Python dependencies
```