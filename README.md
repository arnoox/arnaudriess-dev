# arnaudriess.dev

Static site built with [Zola](https://www.getzola.org) and deployed to GitHub Pages on every push to `main`.

```sh
brew install zola   # once; CI pins the version in .github/workflows/pages.yml
zola serve          # http://127.0.0.1:1111, live reload
zola build          # writes public/
```

| Path | Content |
|---|---|
| `config.toml` | site title, description, social handles, Cal.com, Umami |
| `content/` | text pages in Markdown (Impressum); front matter for the home and `/ffi` pages |
| `templates/index.html` | home page layout and copy |
| `templates/` | Tera templates; `components.html` holds the `email` component |
| `templates/ffi.html` | `/ffi` landing page (draft: noindex, not in sitemap) |
| `sass/assets/main.scss` | design system (logo colors ink `#1B1B1B` / paper `#F6F4F3`, condensed uppercase headlines), compiled to `/assets/main.css` |
| `static/` | copied as-is: logo, favicon, `og.png`, `assets/ffi.css` (FFI-only components) |

Content files are templated: use `{{ <email /> }}` for the contact address.
