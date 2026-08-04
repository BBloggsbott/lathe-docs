# Limitations & Roadmap

!!! warning "Early development"
    Lathe is in early development. APIs, the YAML schema, and runtime behaviour are all subject to change between releases.

## Current limitations

- **Graphs must be acyclic.** There's no support yet for cycles or retry loops within a pipeline. There's no control-flow or conditional node type yet to break out of a cycle, so a loop would just run forever.
- **Only one tool kind.** `LLMNode`s can call tools (see [Concepts § Tools](concepts.md#tools)), but `HttpRequest` is the only tool kind implemented so far, and every declared param is required — there's no optional-param support yet.
- **No multi-turn conversation support.** Each `lathe run` invocation or `/invoke` request is a single pass through the graph starting from fresh state. There's no built-in way to continue a conversation across calls, though an `LLMNode` with tools can call them repeatedly within its own turn.
- **Only `graph_version: V1` is supported.**
- **Two providers only:** `OpenAI` and `LMStudio`.

None of these are permanent restrictions; they reflect where the project is today. Check [GitHub issues](https://github.com/BBloggsbott/lathe/issues) for current status before relying on functionality not listed here.

## Roadmap

Reflects the upstream project's own roadmap and may change without notice; treat it as direction, not a committed schedule.

- [x] Web server: serve a pipeline as an HTTP API endpoint
- [x] Tool call support: let `LLMNode`s invoke tools/functions (currently `HttpRequest` only)
- [ ] Multi-turn conversation support
- [ ] Graph visualizer: render a pipeline's node/edge structure for inspection
- [ ] Additional node types (branching, etc.) and tool kinds
- [ ] Cyclic graph support for retry loops and iterative agents
- [ ] Visual UI: local graph builder and debugger for authoring and stepping through pipelines

## Getting help / following development

- Source code: [github.com/BBloggsbott/lathe](https://github.com/BBloggsbott/lathe)
- Releases: [github.com/BBloggsbott/lathe/releases](https://github.com/BBloggsbott/lathe/releases)
- Issues: [github.com/BBloggsbott/lathe/issues](https://github.com/BBloggsbott/lathe/issues)
