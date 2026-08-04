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
  tools:
  - my-http-tool

- !End
  id: end-node
  label: lathe::nodes::end
  out_pointers:
  - /response

connections:
- from: start-node
  to: llm-node
  label: lathe::nodes::start to My LLM Step
- from: llm-node
  to: end-node
  label: My LLM Step to lathe::nodes::end

tools:
  my-http-tool:
    kind: HttpRequest
    name: my-http-tool
    description: Looks something up over HTTP.
    method: GET
    url: https://example.com/api?q={{query}}
    headers:
      Authorization: Bearer ${MY_API_KEY}
    body: null
    timeout: 5000
    response_type: Json
    params:
      query:
        type: Text
        description: The search query.
```

## Top-level fields

| Field | Type | Description |
|---|---|---|
| `graph_version` | string | Schema version. Only `V1` is currently supported. |
| `name` | string | A human-readable name for the pipeline. |
| `nodes` | list | The graph's nodes. See [Nodes](#nodes). |
| `connections` | list | The graph's directed edges. See [Connections](#connections). |
| `provider_configs` | map | LLM provider credentials and endpoints, keyed by id. See [Provider Configs](#provider-configs). |
| `tools` | map | *(optional)* Tools that `LLMNode`s may call, keyed by id. See [Tools](#tools). |

## YAML tag syntax

Each entry in `nodes` is tagged with `!Start`, `!LLMNode`, or `!End` — this tag selects which node type the following fields belong to. It's YAML's syntax for a tagged union: the tag isn't a comment or a custom type name you choose, it's one of the three fixed node kinds Lathe understands. Every node in the list needs exactly one of these tags.

## Nodes

### `Start`

| Field | Type | Description |
|---|---|---|
| `id` | string | Unique node identifier, referenced by `connections`. |
| `label` | string | Human-readable label. |

Constraints:

- Exactly one `Start` node is required per graph — it's the entry point. See [Validation rules](#validation-rules).
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
| `tools` | list of strings | *(optional)* Ids from the top-level `tools` map that this node's LLM may call. Defaults to an empty list. See [Tools](#tools). |

!!! warning "Unknown tool id panics"
    Every string in `tools` must match a key in the top-level `tools` map. An id that doesn't resolve panics when the graph is materialized, rather than failing gracefully.

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

Each entry is a directed edge from one node to another:

```yaml
connections:
- from: start-node
  to: llm-node
  label: lathe::nodes::start to My LLM Step
```

| Field | Type | Description |
|---|---|---|
| `from` / `to` | string | Must match an `id` in `nodes`. |
| `label` | string | A human-readable description of the edge — purely descriptive, it doesn't affect execution. |

Execution order is derived from the graph's topology (a topological sort), not from the order connections appear in the file. A node with multiple outgoing connections fans out to multiple downstream nodes; a node with multiple incoming connections fans in from multiple upstream nodes. See [Examples § Explainer agent](examples.md#explainer-agent-fan-out) for a worked fan-out pipeline.

## Provider Configs

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

## Tools

The top-level `tools` map declares tools that `LLMNode`s can be given access to, keyed by an arbitrary id that `LLMNode.tools` entries reference:

```yaml
tools:
  geocode-tool:
    kind: HttpRequest
    name: geocode-tool
    description: Look Up City Coordinates in Latitude and Longitude
    method: GET
    url: https://nominatim.openstreetmap.org/search?city={{city}}&format=json
    headers:
      User-Agent: lathe-weather-agent/1.0
    body: null
    timeout: 5000
    response_type: Json
    params:
      city:
        type: Text
        description: City name to geocode, e.g. 'Chennai'
```

Each entry is tagged with `kind`, the same tagged-union pattern as the `nodes` list's `!Start`/`!LLMNode`/`!End` tags but spelled as a `kind` field instead of a YAML tag. `HttpRequest` is currently the only supported kind.

### `HttpRequest`

Issues an HTTP request when called, with the LLM's call arguments substituted into `url`, `headers`, and `body`.

| Field | Type | Description |
|---|---|---|
| `name` | string | Tool name surfaced to the LLM. |
| `description` | string | Description surfaced to the LLM, used to decide when to call the tool. |
| `method` | `GET` \| `POST` \| `PUT` \| `PATCH` \| `DELETE` | HTTP method to issue. |
| `url` | string | Request URL. May contain `{{param_name}}` placeholders — see [Param substitution](#param-substitution). |
| `headers` | map of string to string | *(optional)* Request headers. Values may contain `{{param_name}}` placeholders and `${ENV_VAR}` references. Defaults to empty. |
| `body` | string or `null` | *(optional)* Request body. May contain `{{param_name}}` placeholders. Defaults to `null`. |
| `timeout` | integer (milliseconds) | *(optional)* Request timeout. Defaults to `5000`. |
| `response_type` | `Json` \| `Text` | How to parse the response body before handing it back to the LLM. |
| `params` | map | *(optional)* Named arguments the LLM must supply when calling this tool. Defaults to empty. See below. |

`params` entries declare the arguments the tool accepts:

```yaml
params:
  city:
    type: Text
    description: City name to geocode, e.g. 'Chennai'
```

| Field | Type | Description |
|---|---|---|
| `type` | `Text` \| `Integer` \| `Boolean` \| `Float` | Expected JSON type of the argument. The call fails if the LLM supplies a mismatched type. |
| `description` | string | Surfaced to the LLM as part of the tool's parameter schema. |

!!! note "All params are currently required"
    Every declared param is marked required in the schema surfaced to the LLM, and a call fails if any is missing. There's no way to declare an optional param yet.

### Param substitution

Wherever `{{param_name}}` appears in `url`, a header value, or `body`, it's replaced with the stringified value the LLM supplied for `param_name` at call time. A `{{...}}` placeholder naming something that isn't a declared param is left as-is rather than substituted.

### Env vars in headers

Independently of param substitution, header values may reference `${ENV_VAR}` to pull a secret from the process environment (e.g. an API key you don't want committed to the YAML file):

```yaml
headers:
  Authorization: Bearer ${WEATHER_API_KEY}
```

This is resolved once, when the graph is loaded — a missing env var fails immediately at load time rather than on first call.

## Validation rules

Lathe validates a graph when it's loaded (on by default for `lathe run` and `lathe server`):

- Every connection's `from` and `to` must reference a node that exists in `nodes`.
- Every leaf node (no outgoing connections) must be an `End` node, and every `End` node must be a leaf node.
- A graph must have exactly one `Start` node — none or more than one fails validation.
- Every node other than `Start` must have at least one incoming connection; a node with no incoming connection fails validation.

See [Concepts § Validation](concepts.md#validation) for more on when and why this runs.
