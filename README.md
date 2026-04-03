# Sankofa Engine Developer Documentation

Developer documentation for the [Sankofa Engine](https://github.com/sankofa-labs/sankofa-engine) — a sharded, privacy-preserving financial ledger engine for digital assets.

**Current version:** v0.1.0-alpha

## Local Development

### Prerequisites

- [Hugo extended](https://gohugo.io/installation/) (v0.110.0+)
- [Go](https://golang.org/dl/) (1.21+)
- [Node.js](https://nodejs.org/) (20+)

### Setup

```bash
git clone https://github.com/sankofa-labs/sankofa-engine-dev-docs.git
cd sankofa-engine-dev-docs
npm install
```

### Development Server

```bash
hugo server
```

Open http://localhost:1313 to view the docs locally. Changes to content files are hot-reloaded.

### Build

```bash
hugo --gc --minify
```

Output goes to `public/`.

## Deployment

Deployed automatically to GitHub Pages via GitHub Actions on push to `main`. See `.github/workflows/deploy-docs.yml`.

## Project Structure

- `content/` — Documentation pages (Markdown)
- `static/openapi/` — OpenAPI 3.0 specification
- `static/swagger-ui/` — Swagger UI distribution
- `layouts/` — Hugo templates and shortcodes
- `assets/scss/` — Theme overrides
- `docs/` — Internal specs and plans (not published)

## License

Proprietary — Sankofa Labs Inc.
