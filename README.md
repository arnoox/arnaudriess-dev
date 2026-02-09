# arnaudriess.dev

Professional consulting website for Arnaud Riess — Safety-Critical Rust Consultant.

Built with [Jekyll](https://jekyllrb.com/) 4.4 using a customized [Minima](https://github.com/jekyll/minima) theme, deployed to [GitHub Pages](https://pages.github.com/).

## Tech stack

- **Generator**: Jekyll 4.4.1 (Ruby)
- **Theme**: Minima 2.5 with custom layouts, includes, and SCSS overrides
- **Plugins**: jekyll-feed (RSS), jekyll-seo-tag (Open Graph / Twitter Cards / schema.org)
- **Integrations**: Cal.com booking widget, privacy-respecting Google Analytics
- **Deployment**: GitHub Actions → GitHub Pages (on push to `main`)

## Local development

### Option A — Dev Container (recommended)

1. Open Docker Desktop
2. Open the repo in VS Code → `Cmd+Shift+P` → **Reopen in Container**
3. In the integrated terminal:

```sh
bundle exec jekyll serve --livereload
```

4. Open http://127.0.0.1:4000/

### Option B — Native

Requires Ruby and Bundler installed locally.

```sh
bundle install
bundle exec jekyll serve --livereload
```

## Project structure

```
├── _config.yml            # Site metadata, social links, plugin config
├── _includes/             # Reusable HTML partials (header, footer, email, social icons)
├── _layouts/              # Page templates (default, home, page, post)
├── _sass/                 # SCSS overrides (fonts, layout, syntax highlighting)
├── assets/                # Main stylesheet entry point, social icon sprites
├── index.markdown         # Home page content
├── impressum.markdown     # Legal notice (Impressum)
├── 404.html               # Custom 404 page
├── Gemfile                # Ruby dependencies
├── .devcontainer/         # VS Code dev container config
└── .github/workflows/     # CI/CD pipeline (jekyll.yml)
```

## Deployment

Pushing to `main` triggers the GitHub Actions workflow in [.github/workflows/jekyll.yml](.github/workflows/jekyll.yml), which builds the site and deploys it to GitHub Pages at **https://arnaudriess.dev**.
