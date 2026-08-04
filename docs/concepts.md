# Concepts

This page explains the mental model behind Lathe pipelines. For the exact field-by-field schema, see [Pipeline YAML Reference](pipeline-reference.md).

## Pipelines are graphs

A Lathe pipeline YAML describes a directed **acyclic** graph: a list of **nodes** (the processing steps) and a list of **connections** (directed edges between them), with no cycles i.e., a node can't (directly or indirectly) feed back into itself. When you run a pipeline, Lathe executes the nodes in topological order. The order is implied by the graph's structure and not the order nodes happen to appear in the YAML file. Cycles aren't supported yet mainly because there's no control-flow or conditional node type to break out of one. Without that, a cyclic graph would just loop forever; see [Limitations & Roadmap](limitations.md).

## `AgentState`

Each run threads a single JSON object, the `AgentState`, through the graph. Nodes read fields from it and write fields back to it, addressed by [JSON Pointer](https://datatracker.ietf.org/doc/html/rfc6901) paths such as `/message` or `/output`. The root state must always be a non-empty JSON object.

For example, running the `simple` example pipeline with the message `"Hello!"` starts with:

```json
{ "message": "Hello!" }
```

and after the `LLMNode` runs, becomes:

```json
{ "message": "Hello!", "output_message": "Hi there! How can I help you today?" }
```

This accumulated state is internal to the run. It's not what `lathe run` prints or `/invoke` returns. The `End` node narrows it down to just its `out_pointers` before it comes back to you; see [Node types § End](#node-types) below.

Because state accumulates rather than being replaced, downstream nodes can read anything an upstream node wrote or anything present in the initial state and not just their immediate predecessor's output.

## Node types

Every node in a pipeline is one of three kinds (with more to come). Full field tables are in the [Pipeline YAML Reference](pipeline-reference.md#nodes).

- **`Start`** - the graph's entry point. Exactly one per graph. Passes the initial state through unchanged; errors if the initial state is empty.
- **`LLMNode`** - calls an LLM. Reads a value from state (`input_key`), sends it (along with a `system_prompt`) to the configured provider and model, and writes the response back to state (`output_key`). It may also be given `tools` to call mid-conversation — see [Tools](#tools) below.
- **`End`** - a terminal node. Every node with no outgoing connections must be an `End` node, and vice versa. Declares which state keys to surface as the pipeline's output (`out_pointers`).

## Connections and execution order

A connection links a `from` node id to a `to` node id, plus a `label` — a human-readable description of the edge with no effect on execution.

Execution order comes from the graph's topology, so:

- A node with **multiple outgoing connections** fans out: once it finishes, each of its downstream nodes becomes eligible to run. Those downstream nodes don't depend on each other, only on the shared node that fed them.
- A node with **multiple incoming connections** fans in: it doesn't run until *every* one of its upstream nodes has finished, and it can then read whatever any of them wrote to state — not just one branch's output.

So a fan-out/fan-in shape acts like a join point: several nodes branch off from a common ancestor and run independently, then a later node waits for all of them before reading their combined results. The `explainer` example pipeline uses exactly this shape — two LLM nodes fan out from one explainer node, and the `End` node fans back in to read both of their outputs. See [Examples § Explainer agent](examples.md#explainer-agent-fan-out) for a full walkthrough.

## Provider Configs

The top-level `provider_configs` map holds LLM credentials and endpoints, keyed by an id that `LLMNode`s reference via `provider_config_id`:

```yaml
provider_configs:
  my-openai-config:
    id: my-openai-config
    provider: OpenAI
    api_key: null     # falls back to OPENAI_API_KEY env var
    base_url: null    # null = provider default
```

`base_url: null` resolves to OpenAI's default endpoint for the `OpenAI` provider, or `http://localhost:1234/v1` for `LMStudio`.

!!! note "Unmatched `provider_config_id` falls back silently"
    If an `LLMNode`'s `provider_config_id` doesn't match any key in `provider_configs`, Lathe doesn't error — it silently uses an auto-generated default config for that node's `provider`. This is worth knowing before you spend time debugging why a node is hitting the wrong endpoint: check the id actually matches a key in `provider_configs`.

## Tools

`LLMNode`s can be given tools to call while producing their response. Tools are declared once in the top-level `tools` map and referenced by id from an `LLMNode`'s own `tools` list:

```yaml
tools:
  geocode-tool:
    kind: HttpRequest
    name: geocode-tool
    description: Look Up City Coordinates in Latitude and Longitude
    method: GET
    url: https://nominatim.openstreetmap.org/search?city={{city}}&format=json
    response_type: Json
    params:
      city:
        type: Text
        description: City name to geocode
```

`HttpRequest` is currently the only tool kind: it issues an HTTP request, substituting the LLM's call arguments into the URL, headers, or body wherever `{{param_name}}` appears. Header values can separately reference `${ENV_VAR}` to pull secrets (like an API key) from the environment rather than committing them to the YAML.

Once given tools, the underlying agent can call them repeatedly within a single `LLMNode` invocation — capped at 25 turns — before producing its final response for `output_key`. This is a tool-calling loop within one node's turn, not a multi-turn conversation across separate `lathe run`/`/invoke` calls; see [Current constraints](#current-constraints) below.

See [Pipeline YAML Reference § Tools](pipeline-reference.md#tools) for the full schema, and [Examples § Weather agent](examples.md#weather-agent-tool-calling) for a worked example.

## Template resolution

An `LLMNode`'s `system_prompt` can reference state values with `{{/pointer}}` placeholders, where `pointer` is a JSON Pointer:

```yaml
system_prompt: You are an expert in {{/message}} who summarizes text.
```

!!! tip "Not Jinja or Handlebars"
    This is a small, purpose-built templating syntax. So only `{{/pointer}}` substitution, no filters, conditionals, or loops. Don't reach for Jinja/Handlebars syntax here; it won't work.

Placeholders are resolved against the current `AgentState` immediately before the node calls the LLM. So a placeholder can reference a value written by an earlier node in the graph, not just the initial state. Whitespace inside the braces is trimmed, and non-string values are converted to their string representation.

## Validation

When a pipeline YAML is loaded — the default behavior for `lathe run` and `lathe server` — Lathe validates its structure:

- Every connection must reference node ids that actually exist in the graph.
- Every leaf node (no outgoing connections) must be an `End` node, and every `End` node must be a leaf node.
- A graph must have exactly one `Start` node.
- Every non-`Start` node must have at least one incoming connection — a node no connection ever points to fails validation.

See [Pipeline YAML Reference § Validation rules](pipeline-reference.md#validation-rules) for the authoritative list.

## Current constraints

Lathe's graph engine currently only supports acyclic graphs, tool calling only supports `HttpRequest` tools with required params, and `LLMNode`s can't yet hold a multi-turn conversation across separate runs. See [Limitations & Roadmap](limitations.md) for the full picture.
