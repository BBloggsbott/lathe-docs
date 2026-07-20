# CLI Reference

```
lathe <SUBCOMMAND>
```

| Subcommand | Description |
|---|---|
| [`lathe example`](#lathe-example) | Create an example pipeline YAML file. |
| [`lathe run`](#lathe-run) | Run a pipeline once from a YAML file. |
| [`lathe server`](#lathe-server) | Serve a pipeline as an HTTP API. |

## `lathe example`

```
lathe example <NAME> --provider <PROVIDER> --model <MODEL>
```

| Argument | Flag | Values | Description |
|---|---|---|---|
| `name` | *(positional)* | `simple`, `explainer`, `none` | Which built-in example to generate. |
| `provider` | `-p`, `--provider` | `open-ai`, `lm-studio` | LLM provider written into the generated pipeline's `provider_configs`. |
| `model` | `-m`, `--model` | any string | Model name written into the generated pipeline. |

Creates an `examples/` directory in the current working directory if it doesn't already exist, then writes `examples/simple_agent.yaml` or `examples/explainer_agent.yaml` depending on `name` (`none` does nothing). Both generated files use a `provider_configs` entry with `api_key: null` and `base_url: null`, so credentials come from the environment (see [Environment variables](#environment-variables)) or from LM Studio's default local endpoint.

```sh
lathe example simple --provider open-ai --model gpt-5-mini
```

See [Examples](examples.md) for what each generated pipeline looks like and a walkthrough of running it.

## `lathe run`

```
lathe run --pipeline <PATH> --message <MESSAGE>
```

| Argument | Flag | Required | Description |
|---|---|---|---|
| `pipeline` | `-p`, `--pipeline` | yes | Path to the pipeline YAML file. |
| `message` | `-m`, `--message` | yes | The user message to send into the pipeline. |

Loads and [validates](pipeline-reference.md#validation-rules) the pipeline YAML, sets the initial `AgentState` to `{"message": "<MESSAGE>"}`, runs the pipeline once end to end, and pretty-prints the `End` node's selected output (its `out_pointers`) as JSON to stdout, not the full internal state.

```sh
lathe run --pipeline examples/simple_agent.yaml --message "Hello!"
```

```json
{
  "output_message": "Hi there! How can I help you today?"
}
```

## `lathe server`

```
lathe server --pipeline <PATH> [--host <HOST>] [--port <PORT>]
```

| Argument | Flag | Default | Description |
|---|---|---|---|
| `pipeline` | `-p`, `--pipeline` | *(required)* | Path to the pipeline YAML file. |
| `host` | `-H`, `--host` | `127.0.0.1` | Host to bind the HTTP server to. |
| `port` | `-P`, `--port` | `8080` | Port to bind the HTTP server to. |

!!! note "Uppercase short flags"
    The short forms for `--host` and `--port` are capitalized: `-H` and `-P`, not `-h`/`-p`. This avoids a collision with `-p`/`--pipeline`. `-h` is reserved by clap for `--help`.

Loads and validates the pipeline YAML, then serves an HTTP API:

| Route | Method | Description |
|---|---|---|
| `/health` | `GET` | Liveness check. Returns `{"status": "ok", "pipeline": "<name>"}`. |
| `/invoke` | `POST` | Runs the pipeline once. The request body is used as the initial `AgentState`; it must be a non-empty JSON object. Returns the `End` node's selected output (its `out_pointers`) as JSON, not the full internal state. Returns `400` if the body isn't a non-empty JSON object, `500` if execution fails. |

```sh
lathe server --pipeline examples/simple_agent.yaml --host 127.0.0.1 --port 8080
```

```sh
curl -X POST http://127.0.0.1:8080/invoke \
  -H 'Content-Type: application/json' \
  -d '{"message": "Hello!"}'
```

```json
{
  "output_message": "Hi there! How can I help you today?"
}
```

## Environment variables

| Variable | Used when |
|---|---|
| `OPENAI_API_KEY` | An `OpenAI` entry in `provider_configs` has `api_key: null`. |

Lathe automatically loads a `.env` file from the current working directory on startup (via [dotenvy](https://github.com/allan2/dotenvy)), so you can put `OPENAI_API_KEY=sk-...` there instead of exporting it in your shell.
