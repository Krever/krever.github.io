# w.pitula.me

My personal home page — plain static HTML, no build step.

- `index.html` — the home page (self-contained: HTML + inline CSS + inline SVG icons).
- `CNAME` — custom domain (`w.pitula.me`).
- `blog-archive/` — frozen copies of the old Jekyll blog articles (2016–2018). **Not
  linked from the home page.** The old `/<year>/<slug>/` URLs still resolve via redirect
  stubs in `2016/ 2017/ 2018/`.
- `css/ js/ fonts/ images/ feed.xml` — assets referenced by the archived articles (and
  favicons used by the home page).
- `.github/workflows/deploy.yml` — on every push to `master`, publishes the repo to
  GitHub Pages (no build).

## Deploy

Just push to `master`. GitHub Actions deploys automatically.

> One-time setup: GitHub → repo **Settings → Pages → Source → "GitHub Actions"**.

## Subprojects

`w.pitula.me/presentations`, `w.pitula.me/fintech-engineering-handbook`, etc. are
**separate repositories** served under this domain because this repo is the
`krever.github.io` user-pages site. They are independent of this repo's content.
