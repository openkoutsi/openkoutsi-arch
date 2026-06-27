# openkoutsi-arch

Architectural documentation for the **openkoutsi** platform — a self-hosted cycling coaching
system. The site is built with [MkDocs](https://www.mkdocs.org/) +
[Material](https://squidfunk.github.io/mkdocs-material/) and published to GitHub Pages.

**📖 Read the docs: <https://openkoutsi.github.io/openkoutsi-arch/>**

It describes the **target v2 architecture**: a single-instance, per-user database model with a
simplified, token-scoped API.

> This is an **architecture reference** for contributors and operators. End-user documentation
> lives in [openkoutsi/openkoutsi-docs](https://github.com/openkoutsi/openkoutsi-docs); the
> code lives in [openkoutsi/openkoutsi-backend](https://github.com/openkoutsi/openkoutsi-backend)
> (backend) and [openkoutsi/openkoutsi-web](https://github.com/openkoutsi/openkoutsi-web)
> (frontend).

## Build locally

Requires [uv](https://docs.astral.sh/uv/).

```console
uv sync
uv run mkdocs serve     # live preview at http://127.0.0.1:8000
```

Before pushing, build strictly — this is exactly what CI checks on pull requests:

```console
uv run mkdocs build --strict
```

`--strict` fails on broken links and missing nav entries. Fix any warnings before opening a PR.

## Deployment

`main` is deployed automatically to GitHub Pages by `.github/workflows/deploy.yml`.
Pull requests and non-`main` pushes are validated by `.github/workflows/ci.yml` (strict build).

The Pages source must be set to **GitHub Actions** once in the repository settings
(Settings → Pages → Build and deployment → Source: GitHub Actions).

## License

Apache-2.0. See [LICENSE](LICENSE).
