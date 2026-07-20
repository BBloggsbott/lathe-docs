# Examples

Lathe ships two built-in example pipelines, generated with `lathe example <name>`. This page walks through both.

## Simple agent

Generate it with:

```sh
lathe example simple --provider open-ai --model gpt-5-mini
```

This writes `examples/simple_agent.yaml`:

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
  provider_config_id: my-model
- !End
  id: end-node
  label: lathe::nodes::end
  out_pointers:
  - /output_message
connections:
- from:
    node_id: start-node
    name: to Simple Assistant LLM Node
  to:
    node_id: llm-node
    name: from lathe::nodes::start
- from:
    node_id: llm-node
    name: to lathe::nodes::end
  to:
    node_id: end-node
    name: from Simple Assistant LLM Node
provider_configs:
  my-model:
    id: my-model
    base_url: null
    api_key: null
    provider: OpenAI
```

The graph shape is a straight line:

```mermaid
graph LR
    A[start-node] --> B[llm-node]
    B --> C[end-node]
```

The LLM node reads `/message` (the input you pass on the command line), sends it to the model with the system prompt "You are a helpful assistant", and writes the response to `/output_message`, which the `End` node surfaces as output.

```sh
lathe run --pipeline examples/simple_agent.yaml --message "Hello!"
```

```json
{
  "output_message": "Hi there! How can I help you today?"
}
```

Only `/output_message` appears as `lathe run` prints the `End` node's selected output (`out_pointers`), not the full internal state. So `/message` isn't included even though it was in the initial state.

## Explainer agent (fan-out)

Generate it with:

```sh
lathe example explainer --provider open-ai --model gpt-5-mini
```

This writes `examples/explainer_agent.yaml`, a five-node pipeline:

```yaml
graph_version: V1
name: Example Lathe Graph - Explainer
nodes:
- !Start
  id: start-node
  label: lathe::nodes::start
- !LLMNode
  id: llm-explainer-node
  label: Explainer LLM Node
  provider: OpenAI
  model: gpt-5-mini
  system_prompt: You are a knowledgeable assistant who explains topics clearly. For the given question or topic, provide a detailed, well-structured explanation that builds from foundational concepts to more nuanced points, using concrete examples where helpful.
  input_key: /message
  output_key: /explanation
  provider_config_id: my-model
- !LLMNode
  id: llm-summarizer-node
  label: Explainer LLM Node
  provider: OpenAI
  model: gpt-5-mini
  system_prompt: You are an expert in {{/message}} who summarizes text. Given the text, produce a concise summary of two to three sentences that captures the key points while preserving the original meaning.
  input_key: /explanation
  output_key: /summary
  provider_config_id: my-model
- !LLMNode
  id: llm-topic-generator-node
  label: Explainer LLM Node
  provider: OpenAI
  model: gpt-5-mini
  system_prompt: You are an expert in {{/message}} who writes titles for text. Given some text, generate a short, descriptive title (five words or fewer) that captures its essence.
  input_key: /explanation
  output_key: /title
  provider_config_id: my-model
- !End
  id: end-node
  label: lathe::nodes::end
  out_pointers:
  - /explanation
  - /summary
  - /title
connections:
- from:
    node_id: start-node
    name: to Explainer LLM Node
  to:
    node_id: llm-explainer-node
    name: from lathe::nodes::start
- from:
    node_id: llm-explainer-node
    name: to Explainer LLM Node
  to:
    node_id: llm-summarizer-node
    name: from Explainer LLM Node
- from:
    node_id: llm-explainer-node
    name: to Explainer LLM Node
  to:
    node_id: llm-topic-generator-node
    name: from Explainer LLM Node
- from:
    node_id: llm-summarizer-node
    name: to lathe::nodes::end
  to:
    node_id: end-node
    name: from Explainer LLM Node
- from:
    node_id: llm-topic-generator-node
    name: to lathe::nodes::end
  to:
    node_id: end-node
    name: from Explainer LLM Node
provider_configs:
  my-model:
    id: my-model
    base_url: null
    api_key: null
    provider: OpenAI
```

The graph shape:

```mermaid
graph LR
    A[start-node] --> B[llm-explainer-node]
    B --> C[llm-summarizer-node]
    B --> D[llm-topic-generator-node]
    C --> E[end-node]
    D --> E[end-node]
```

`llm-explainer-node` reads the initial `/message` and writes a long explanation to `/explanation`. From there the graph **fans out**: both `llm-summarizer-node` and `llm-topic-generator-node` read `/explanation` and run independently. One writes `/summary`, the other `/title`. The `End` node then **fans in**, surfacing all three: `/explanation`, `/summary`, and `/title`.

Notice that both downstream nodes also template `{{/message}}` into their system prompts (`"You are an expert in {{/message}} who summarizes text..."`) even though `/message` isn't their `input_key`. This works because `AgentState` accumulates. The original message written by `Start` is still there for any later node to read, not just the node it was originally passed to. See [Concepts § AgentState](concepts.md#agentstate) and [§ Template resolution](concepts.md#template-resolution).

```sh
lathe run --pipeline examples/explainer_agent.yaml --message "quantum entanglement"
```

```json
{
  "explanation": "Quantum entanglement is a phenomenon where two or more particles become linked...",
  "summary": "Quantum entanglement links particles so that measuring one instantly affects the other...",
  "title": "Understanding Quantum Entanglement"
}
```

`/message` isn't in the output even though the summarizer and title-generator nodes read it via `{{/message}}` templating. Only the three pointers listed in the `End` node's `out_pointers` (`/explanation`, `/summary`, `/title`) are surfaced.

## Serving either example

Both examples work the same way with `lathe server`. The pipeline doesn't change, only how you invoke it:

```sh
lathe server --pipeline examples/explainer_agent.yaml --port 8080
```

```sh
curl -X POST http://127.0.0.1:8080/invoke \
  -H 'Content-Type: application/json' \
  -d '{"message": "quantum entanglement"}'
```

See also: [Pipeline YAML Reference](pipeline-reference.md) for the full schema, and [CLI Reference](cli-reference.md) for every flag.
