# Getting Started

This page walks you through installing Lathe, generating an example pipeline, and running it.

## Prerequisites

- An LLM provider: either an OpenAI API key, or a running [LM Studio](https://lmstudio.ai/) instance.
- A Rust toolchain (stable), **only** if you plan to install via `cargo install` or build from source rather than downloading a prebuilt binary.

## Installation

Pick one of the following.

=== "Prebuilt binary"

    Download the binary for your platform from the [latest release](https://github.com/BBloggsbott/lathe/releases/latest):

    | Platform | Asset |
    |----------|-------|
    | Linux (x86_64) | `lathe-linux-x86_64` |
    | macOS (Intel) | `lathe-macos-x86_64` |
    | macOS (Apple Silicon) | `lathe-macos-aarch64` |
    | Windows (x86_64) | `lathe-windows-x86_64.exe` |

    On Linux or macOS, substitute the asset name for your platform from the table above (this example uses Linux x86_64) — the `-o lathe` flag saves it locally as `lathe`, without the platform suffix:

    ```sh
    curl -L -o lathe https://github.com/BBloggsbott/lathe/releases/latest/download/lathe-linux-x86_64
    chmod +x lathe
    sudo mv lathe /usr/local/bin/
    ```

    On Windows, download `lathe-windows-x86_64.exe` and place it somewhere on your `PATH`.

=== "cargo install"

    `lathe-cli` is published on [crates.io](https://crates.io/crates/lathe-cli). With a stable Rust toolchain installed:

    ```sh
    cargo install lathe-cli
    ```

    This builds an optimized binary and places it on your `PATH` as `lathe`.

=== "Build from source"

    ```sh
    cargo build --release
    # binary at target/release/lathe (target/release/lathe.exe on Windows)
    ```

    Or run it directly without installing:

    ```sh
    cargo run -p lathe-cli -- <args>
    ```

## Verify the install

```sh
lathe --help
```

You should see the three subcommands: `example`, `run`, and `server`.

## Set up your LLM provider

If you're using OpenAI, set your API key as an environment variable or in a `.env` file in your working directory (Lathe loads `.env` automatically on startup):

```sh
export OPENAI_API_KEY=sk-...
```

If you're using LM Studio instead, start LM Studio's local server — Lathe talks to it at `http://localhost:1234/v1` by default, no API key required. See [Concepts § Provider Configs](concepts.md#provider-configs) for how to point at a different endpoint.

## Quickstart: your first pipeline

Generate the built-in `simple` example, which asks an LLM to answer a message and prints its response:

```sh
lathe example simple --provider open-ai --model gpt-5-mini
```

This creates an `examples/` directory (if it doesn't already exist) containing `examples/simple_agent.yaml` — a three-node Start → LLM → End pipeline. See it broken down in [Examples](examples.md#simple-agent).

Run it:

```sh
lathe run --pipeline examples/simple_agent.yaml --message "Hello!"
```

Lathe pretty-prints the pipeline's output as JSON — just the keys the `End` node's `out_pointers` selects (here, `/output_message`), not the full internal state:

```json
{
  "output_message": "Hi there! How can I help you today?"
}
```

The exact text will vary — it comes from the model you configured.

## Serving a pipeline over HTTP

Instead of running a pipeline once from the CLI, you can serve it as an HTTP API:

```sh
lathe server --pipeline examples/simple_agent.yaml --host 127.0.0.1 --port 8080
```

Then, from another terminal:

```sh
curl -X POST http://127.0.0.1:8080/invoke \
  -H 'Content-Type: application/json' \
  -d '{"message": "Hello!"}'
```

The full route reference, including `GET /health`, is in [CLI Reference § lathe server](cli-reference.md#lathe-server).

## Next steps

- [Concepts](concepts.md) — understand how graphs, state, and templating work.
- [Pipeline YAML Reference](pipeline-reference.md) — write your own pipeline from scratch.
- [Examples](examples.md) — see a more advanced, fan-out pipeline.
