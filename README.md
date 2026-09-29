# kenan.sh

Source for [kenan.sh](https://kenan.sh), my blog and personal corner of the web: posts, notes,
homelab write-ups, photography and experiments. Built with [Hugo](https://gohugo.io/) and a
customised, vendored copy of the [typo](https://github.com/tomfran/typo) theme (`themes/typo`,
with project overrides in `layouts/` and `assets/css/`).

My portfolio and CV live separately at [kenanjasim.com](https://kenanjasim.com).

## Development

```bash
hugo server
```

## Deployment

GitHub Actions builds the site with Hugo and deploys it to GitHub Pages on every push to `main`
(`.github/workflows/deploy.yaml`); other branches get a CI build only (`ci.yaml`). The custom
domain `kenan.sh` is set in the repository's Pages settings, with DNS on Cloudflare.

Until September 2026 this site lived at kenanjasim.com. Old URLs are 301-redirected to the same
path on kenan.sh by a Cloudflare Redirect Rule on the kenanjasim.com zone.
