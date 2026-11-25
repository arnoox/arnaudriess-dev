# SEO to-do (temporary)

Temporary checklist for making the Zola site indexable. Delete this file once
everything is done.

Status on 2026-11-21: the site only runs as the hidden preview
(`/preview-377cf0ec9ccb/`, noindex injected by `jekyll.yml` on `main`). The live
site is still Jekyll.

## Blockers: fix before going live

- [ ] **Rust ↔ C page is hidden from Google.** `content/rust-c-interop.md` has
      `noindex = true` and `sitemap = false`. Remove both on go-live day.
- [x] **Example post is published.** Done: renamed to `2026-11-20-hello-blog-draft.md`, `draft = true`. `content/blog/2026-11-20-hello-blog.md` is
      not a draft and appears in `sitemap.xml` with placeholder text. Delete it
      or set `draft = true`.
- [ ] **Blog index copies the home page's SEO.** `templates/base.html` only checks
      `page`, never `section`. `/blog/` gets the home title, description and
      `canonical = https://arnaudriess.dev/`, so Google sees a duplicate of the
      home page. Add a `section` branch (title, description, permalink).
- [ ] **Home title too long (~170 characters).** Google truncates at ~60 and
      the Rust ↔ C keyword is missing. Proposal:
      "Safety-critical Rust and Rust ↔ C interop · Arnaud Riess".
- [ ] **Impressum** uses the default site description. Give it its own
      description, or `noindex`.

## Improvements

- [ ] JSON-LD `Person` / `ProfessionalService`: name, Stuttgart, languages,
      `sameAs` LinkedIn + GitHub (from `config.extra`).
- [ ] JSON-LD `Service` + `Offer` on the Rust ↔ C page, prices from
      `config.extra.prices` (single source of truth stays intact).
- [ ] Blog posts: `og:type = article`, JSON-LD `BlogPosting` with
      `datePublished` / `dateModified`.
- [ ] OG image 1200×630 instead of the square 1024×1024 `static/assets/og.png`,
      and `twitter:card = summary_large_image`. Optional: per-page image via
      `extra.og_image`.
- [ ] Sitemap `<lastmod>`: set `updated` on pages so it is filled in.

## Go-live day (not code)

- [ ] Replace the preview step in `jekyll.yml` with a real Zola deploy. The
      noindex injection only exists in that preview step, so it disappears too.
- [ ] Do the first blocker above (remove `noindex` / `sitemap = false`).
- [ ] Check that the live pages have no `<meta name="robots" content="noindex">`
      (`curl -s https://arnaudriess.dev/ | grep -i robots`).
- [ ] Google Search Console: verify the domain, submit `sitemap.xml`, request
      indexing for `/` and `/rust-c-interop/`.
- [ ] Old URLs: only `/impressum/` existed on Jekyll and keeps its path. `/ffi`
      already redirects to `/rust-c-interop/`. Nothing else to do.

## Content (ongoing)

- [ ] 3 to 5 articles that target search terms, e.g. "Rust C interop ISO 26262",
      "bindgen pitfalls", "Rust FFI safety argument". Link them from LinkedIn posts.
- [ ] Link the site from the LinkedIn profile and GitHub profile.

## Already in place

Title, meta description, canonical, Open Graph, Twitter tags, basic JSON-LD,
`sitemap.xml` (respects `extra.sitemap = false`), `robots.txt`, 404 page,
cookie-free analytics (Umami).
