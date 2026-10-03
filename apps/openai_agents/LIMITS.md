# OpenAI Agents SDK: what each path delivers

Checked against run `oa-20261002T061830` (openai-agents 0.22.3, openai 3.23.0,
openinference-instrumentation-openai-agents 2.5.2, opentelemetry-sdk 1.45.0; model
nvidia/nemotron-3.5-lightning:free).

The SDK traces every run through its own tracing API and by default exports the traces to the OpenAI
platform. The OpenInference instrumentor replaces that exporter with its own processor, so no trace
left the machine. The app adds a second processor that writes the SDK's own spans as they end.

## Spans (OpenInference instrumentor)

The instrumentor turns each SDK span into one OpenTelemetry span with the same start time.

- One AGENT span per agent run, with the agent name. The agent that received a handoff also has
  `graph.node.parent_id`, the agent that handed off.
- One LLM span per model call: model name, `llm.system`, invocation parameters (temperature, max
  tokens, parallel tool calls, base URL), prompt and completion tokens, and the input and output
  messages. `llm.system` is `openai`, the client library; OpenRouter appears only as the base URL. The
  output messages carry the id of every tool call the model requested, and the input messages carry
  the id of every tool result the model read.
- One TOOL span per tool call: tool name, description, parameter schema, arguments, and result. A
  failed call has status ERROR and the error message.
- One TOOL span per handoff, named `handoff to writer`, with the two agent names as input and output.
- The SDK's turn spans, run spans, and the application's custom spans arrive as CHAIN spans with
  their name only.

## Only the SDK's own spans

- The turn number of each step within an agent's loop. Turn numbers continue across a handoff: the
  writer's first turn after the handoff is turn 3.
- The tools and handoffs each agent could call, which give the candidates of a routing.
- The handoff as its own span with `from_agent` and `to_agent`.
- The data of the application's custom spans.
- `group_id`, the conversation the run belongs to.
- A tool call held for approval: a function span with no output and no error.

## Only the SDK's agent objects and run hooks

- Which tools are agents. The trace names a tool such as `weather_agent` but does not say that it
  runs an agent; `get_function_tool_origin` on the agent object does (`agent_as_tool`).
- The cost of each model call. With `ModelSettings(preserve_raw_usage=True)` the raw usage that
  OpenRouter returns, cost included, reaches `RunHooks.on_llm_end`. Agents used as tools run with their
  own hooks, so the hooks are passed to `as_tool` as well.
- The id of the tool call a person approved, from the `ToolApprovalItem`.

## Declared by the application through custom spans

The SDK records these only when the application opens a `custom_span` for them:

- reads and writes of the `notes` memory, with the written value;
- each evaluator verdict and its reason (the evaluator loop is application code, so the trace shows
  separate runs and no loop);
- the approval decision of the person;
- the target of the effect.

## Neither path

- The tool call id on the tool span. The converter pairs each tool span with the model's requested
  calls of the same name in the same turn, and the call approved by a person with the held call of the
  same name and arguments.
- The exception type of a failed tool call: the SDK reports the message only.
- The retry of a failed tool call. The SDK returns the error to the model, which calls the tool
  again; the record links the two calls because they have the same tool and the same arguments.
- What a tool result came from. The result of an agent used as a tool is linked to that agent's
  final answer because their contents are equal (knownBy inferred).
- The time a memory record's content is about.
