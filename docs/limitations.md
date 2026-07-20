# Limitations & Roadmap

!!! warning "Early development"
    Lathe is in early development. APIs, the YAML schema, and runtime behaviour are all subject to change between releases.

## Current limitations

- **Graphs must be acyclic.** There's no support yet for cycles or retry loops within a pipeline. There's no control-flow or conditional node type yet to break out of a cycle, so a loop would just run forever.
- **No tool/function calling.** `LLMNode`s can't yet invoke tools or functions as part of a run.
- **No multi-turn conversation support.** Each `lathe run` invocation or `/invoke` request is a single pass through the graph starting from fresh state. There's no built-in way to continue a conversation across calls.
- **Only `graph_version: V1` is supported.**
- **Two providers only:** `OpenAI` and `LMStudio`.

None of these are permanent restrictions; they reflect where the project is today. Check [GitHub issues](https://github.com/BBloggsbott/lathe/issues) for current status before relying on functionality not listed here.

## Roadmap

Reflects the upstream project's own roadmap and may change without notice; treat it as direction, not a committed schedule.

- [x] Web server: serve a pipeline as an HTTP API endpoint
- [ ] Tool call support: let `LLMNode`s invoke tools/functions
- [ ] Multi-turn conversation support
- [ ] Graph visualizer: render a pipeline's node/edge structure for inspection
- [ ] Additional node types (branching, tool calls, etc.)
- [ ] Cyclic graph support for retry loops and iterative agents
- [ ] Visual UI: local graph builder and debugger for authoring and stepping through pipelines

## Getting help / following development

- Source code: [github.com/BBloggsbott/lathe](https://github.com/BBloggsbott/lathe)
- Releases: [github.com/BBloggsbott/lathe/releases](https://github.com/BBloggsbott/lathe/releases)
- Issues: [github.com/BBloggsbott/lathe/issues](https://github.com/BBloggsbott/lathe/issues)
