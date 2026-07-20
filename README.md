# lathe-docs

User documentation for [lathe](https://github.com/BBloggsbott/lathe), a YAML-based AI agent pipeline builder.

Built with [MkDocs](https://www.mkdocs.org/) and the [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) theme, with versioned deploys handled by [mike](https://github.com/jimporter/mike).

## Contents

- [`getting-started.md`](docs/getting-started.md) — install Lathe and run your first pipeline
- [`concepts.md`](docs/concepts.md) — the mental model: graphs, state, node types, and templating
- [`pipeline-reference.md`](docs/pipeline-reference.md) — every field in the pipeline schema
- [`cli-reference.md`](docs/cli-reference.md) — every `lathe` subcommand and flag
- [`examples.md`](docs/examples.md) — a walkthrough of the two built-in example pipelines
- [`limitations.md`](docs/limitations.md) — what Lathe doesn't do yet

## Local development

This project uses [uv](https://docs.astral.sh/uv/) to manage its Python environment (Python 3.12+).

Install dependencies:

```sh
uv sync
```

Serve the site locally with live reload:

```sh
uv run mkdocs serve
```

The site will be available at `http://127.0.0.1:8000`.

Build a static copy to `site/`:

```sh
uv run mkdocs build
```

## Versioned deploys

Docs are versioned with `mike`. To build and deploy a version:

```sh
uv run mike deploy --push --update-aliases <version> latest
```

## License

Documentation content is licensed under [CC BY 4.0](LICENSE).
