# Pipeline YAML Reference

This page documents every field in the pipeline YAML format. For the concepts behind these fields (state, execution order, templating), see [Concepts](concepts.md).

A full example:

```yaml
graph_version: V1
name: My Agent

provider_configs:
  my-openai-config:
    id: my-openai-config
    provider: OpenAI
    api_key: null        # falls back to OPENAI_API_KEY env var
    base_url: null        # null = default OpenAI endpoint

nodes:
- !Start
  id: start-node
  label: lathe::nodes::start

- !LLMNode
  id: llm-node
  label: My LLM Step
  provider: OpenAI
  model: gpt-4o-mini
  system_prompt: You are a helpful assistant. The user's name is {{/user_name}}.
  input_key: /message
  output_key: /response
  provider_config_id: my-openai-config

- !End
  id: end-node
  label: lathe::nodes::end
  out_pointers:
  - /response

connections:
- from:
    node_id: start-node
    name: to My LLM Step
  to:
    node_id: llm-node
    name: from lathe::nodes::start
- from:
    node_id: llm-node
    name: to lathe::nodes::end
  to:
    node_id: end-node
    name: from My LLM Step
```

## Top-level fields

| Field | Type | Description |
|---|---|---|
| `graph_version` | string | Schema version. Only `V1` is currently supported. |
| `name` | string | A human-readable name for the pipeline. |
| `nodes` | list | The graph's nodes. See [Nodes](#nodes). |
| `connections` | list | The graph's directed edges. See [Connections](#connections). |
| `provider_configs` | map | LLM provider credentials and endpoints, keyed by id. See [provider_configs](#provider_configs). |

## YAML tag syntax

Each entry in `nodes` is tagged with `!Start`, `!LLMNode`, or `!End` — this tag selects which node type the following fields belong to. It's YAML's syntax for a tagged union: the tag isn't a comment or a custom type name you choose, it's one of the three fixed node kinds Lathe understands. Every node in the list needs exactly one of these tags.

## Nodes

### `Start`

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique node identifier, referenced by `connections`. |
| `label` | string | Human-readable label. |

Constraints:

- Exactly one `Start` node is required per graph — it's the entry point.
- It passes the initial `AgentState` through unchanged. If the initial state is empty, it errors.

### `LLMNode`

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique node identifier, referenced by `connections`. |
| `label` | string | Human-readable label. |
| `provider` | `OpenAI` \| `LMStudio` | Which LLM provider to call. |
| `model` | string | Model name to request from the provider. |
| `system_prompt` | string | System prompt sent to the model. Supports `{{/pointer}}` templating — see [Concepts § Template resolution](concepts.md#template-resolution). |
| `input_key` | JSON Pointer | Where to read the user message from in `AgentState`. |
| `output_key` | JSON Pointer | Where to write the LLM's response in `AgentState`. |
| `provider_config_id` | string | Id of an entry in the top-level `provider_configs` map to use for this node. |

!!! note "Unmatched `provider_config_id` falls back silently"
    If `provider_config_id` doesn't match any key in `provider_configs`, the node does **not** error — it silently falls back to an auto-generated default config for the node's `provider` (default endpoint, no API key set explicitly, so it still relies on `OPENAI_API_KEY` for OpenAI). If a node seems to be hitting the wrong endpoint or credentials, double-check this id matches a real key in `provider_configs`.

### `End`

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique node identifier, referenced by `connections`. |
| `label` | string | Human-readable label. |
| `out_pointers` | list of JSON Pointers | Which keys from the final `AgentState` to surface as the pipeline's output. |

Constraints:

- Every leaf node (a node with no outgoing connections) must be an `End` node, and every `End` node must be a leaf node. A graph that violates this fails validation.

## Connections

Each entry connects one node's output port to another node's input port:

```yaml
connections:
- from:
    node_id: start-node
    name: to My LLM Step
  to:
    node_id: llm-node
    name: from lathe::nodes::start
```

| Field | Type | Description |
|---|---|---|
| `from.node_id` / `to.node_id` | string | Must match an `id` in `nodes`. |
| `from.name` / `to.name` | string | A human-readable port label — purely descriptive, it doesn't affect execution. |

Execution order is derived from the graph's topology (a topological sort), not from the order connections appear in the file. A node with multiple outgoing connections fans out to multiple downstream nodes; a node with multiple incoming connections fans in from multiple upstream nodes. See [Examples § Explainer agent](examples.md#explainer-agent-fan-out) for a worked fan-out pipeline.

## `provider_configs`

A map from an arbitrary id to a provider configuration:

```yaml
provider_configs:
  my-openai-config:
    id: my-openai-config
    provider: OpenAI
    api_key: null
    base_url: null
```

| Field | Type | Description |
|---|---|---|
| `id` | string | Must match the map key. Referenced by `LLMNode.provider_config_id`. |
| `provider` | `OpenAI` \| `LMStudio` | Which provider this config is for. |
| `api_key` | string or `null` | API key. For `OpenAI`, `null` falls back to the `OPENAI_API_KEY` environment variable. `LMStudio` doesn't require a key. |
| `base_url` | string or `null` | API endpoint. `null` uses the provider's default: OpenAI's standard endpoint for `OpenAI`, or `http://localhost:1234/v1` for `LMStudio`. |

## Validation rules

Lathe validates a graph when it's loaded (on by default for `lathe run` and `lathe server`):

- Every connection's `node_id` (both `from` and `to`) must reference a node that exists in `nodes`.
- Every leaf node (no outgoing connections) must be an `End` node, and every `End` node must be a leaf node.

See [Concepts § Validation](concepts.md#validation) for more on when and why this runs.
