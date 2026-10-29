# geng-language.github.io

Source for the Geng website, built with [Zola](https://www.getzola.org/).

## Local development

Zola is provided by [Devbox](https://www.jetify.com/devbox) (pinned in `devbox.json` to the same version CI uses):

```sh
devbox run serve    # http://127.0.0.1:1111 with live reload
devbox run serve-lan  # same, but reachable from other hosts at http://<this-host-ip>:1111
devbox run build    # output in public/
devbox run check    # build and check links
```

Or run `devbox shell` and use `zola` directly. When upgrading Zola, update both
`devbox.json` and `ZOLA_VERSION` in `.github/workflows/deploy.yml`.

## Layout

| Path | Purpose |
|---|---|
| `content/docs/` | Guides and tutorials |
| `content/reference/` | Language reference |
| `content/blog/` | Blog posts (`YYYY-MM-DD-slug.md`) |
| `templates/` | Tera templates |
| `sass/site.scss` | Site styles |
| `syntaxes/geng.tmLanguage.json` | TextMate grammar for ` ```geng ` code blocks |

Pages in `docs/` and `reference/` are ordered by the `weight` front-matter field.

## Logo font

The header wordmark and the favicon are drawn from the Feelfree font by Billy
Argel. It is free for personal use only; a commercial license is sold at
<https://billyargel.com/product/feelfree/>. Only the glyph outlines are in the
repo (`templates/partials/wordmark.html`, `static/logos/favicon.svg`), not the
font file.

## Deployment

Pushes to `main` are built and deployed by `.github/workflows/deploy.yml`.
In the repo settings, set **Pages → Source** to **GitHub Actions**.

For a custom domain, set it under **Pages → Custom domain**. The workflow picks
up the resulting base URL automatically.
