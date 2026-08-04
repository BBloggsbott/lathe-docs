# Lathe

Lathe is a YAML-based AI agent builder. You define agent pipelines as directed graphs in YAML, and run them from the CLI or serve them over HTTP.

!!! warning "Early development"
    Lathe is in early development. APIs, the YAML schema, and runtime behaviour are all subject to change between releases. Check the [GitHub releases](https://github.com/BBloggsbott/lathe/releases) page before upgrading, and see [Limitations & Roadmap](limitations.md) for what's not built yet.

## What you can build

A Lathe pipeline is a graph of **nodes** connected by **connections**. A graph typically consists of a starting point, one or more steps that call an LLM or perform other operations, and an ending point that declares the output. Each run threads a JSON object (the `AgentState`) through the graph, with nodes reading and writing fields by [JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) path. You can run a pipeline once from the command line, or stand it up as an HTTP API with a single command.

Here's a minimal Start -> LLM -> End graph where the LLM node can call a tool:

```yaml
graph_version: V1
name: Example Lathe Graph - Simple
nodes:
- !Start
  id: start-node
  label: lathe::nodes::start
- !LLMNode
  id: llm-node
  label: Simple Assistant LLM Node
  provider: OpenAI
  model: gpt-5-mini
  system_prompt: You are a helpful assistant
  input_key: /message
  output_key: /output_message
  provider_config_id: gpt-5-mini
  tools:
  - quote-tool
- !End
  id: end-node
  label: lathe::nodes::end
  out_pointers:
  - /output_message
connections:
- from: start-node
  to: llm-node
  label: lathe::nodes::start to Simple Assistant LLM Node
- from: llm-node
  to: end-node
  label: Simple Assistant LLM Node to lathe::nodes::end
provider_configs:
  gpt-5-mini:
    id: gpt-5-mini
    base_url: null
    api_key: null
    provider: OpenAI
tools:
  quote-tool:
    kind: HttpRequest
    name: quote-tool
    description: Get an inspirational quote about a given topic
    method: GET
    url: https://api.example.com/quotes?topic={{topic}}
    response_type: Json
    params:
      topic:
        type: Text
        description: Topic to fetch a quote about, e.g. 'perseverance'
```

## Where to go next

- **[Getting Started](getting-started.md)** — install Lathe and run your first pipeline.
- **[Concepts](concepts.md)** — the mental model: graphs, state, node types, and templating.
- **[Pipeline YAML Reference](pipeline-reference.md)** — every field in the pipeline schema.
- **[CLI Reference](cli-reference.md)** — every `lathe` subcommand and flag.
- **[Examples](examples.md)** — a walkthrough of the three built-in example pipelines.
- **[Limitations & Roadmap](limitations.md)** — what Lathe doesn't do yet.

Lathe is open source under the GPL-3.0 license. Source code: [github.com/BBloggsbott/lathe](https://github.com/BBloggsbott/lathe).
