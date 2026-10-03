# Strands Agents: what each path delivers

Checked against run `st-20261002T082412` (strands-agents 1.57.2, openai 2.54.0, opentelemetry-sdk
1.45.0; model cohere/north-mini-code:free through OpenRouter). Strands raises on an answer cut at the
output token limit, where the other frameworks return it.

Strands writes OpenTelemetry spans itself through the global tracer provider; no instrumentor is
needed and no exporter was configured, so no span left the machine. The app sets
`OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental,gen_ai_span_attributes_only`, which
selects the latest GenAI conventions and keeps message content in span attributes.

## Spans (Strands' built-in tracing)

- One `invoke_graph` span per graph run and one `invoke_swarm` span per swarm run, with the input
  messages. Neither names the graph, its nodes, or its edges.
- One `invoke_agent` span per agent run: agent name, request model, system prompt, input and output
  messages, and token counts summed over the run.
- One `execute_event_loop_cycle` span per turn of an agent's loop, with a `event_loop.cycle_id`.
- One `chat` span per model call: request model, input and output messages, token counts, and time
  to first token. The finish reason is only inside the output messages. There is no temperature, max
  tokens, response model, or cost. `gen_ai.provider.name` is `strands-agents`, the framework; the
  service that answered, OpenRouter, is not named.
- One `execute_tool` span per tool call: tool name, call id, description, schema, arguments, result,
  and `gen_ai.tool.status`. The model's input and output messages carry the same call ids, so model
  calls and tool calls link by id.
- The swarm handoff as an ordinary `execute_tool handoff_to_agent` span, whose arguments name the
  target agent and carry the message.
- `memory.add`, `memory.inject`, and `memory.search` spans from the memory manager, with the store
  names, the result count, and the content in a bare `content` attribute. On `memory.search` the
  results overwrite the query under that key, so the query is lost.
- A run interrupted for approval: the held tool call has a span with status OK and no result, and the
  interrupted agent's output message is the interrupt itself (id, name, and reason), written as a
  Python list.
- `session.id` and `user.id` only because the app passes them as `trace_attributes`.

## Only the native hooks and objects

- Which graph or swarm node each agent run belongs to: `BeforeNodeCallEvent` gives the node id, and
  the graph object gives the nodes, the agent behind each node, and the edges. The edges give the
  candidates of the routing; the nodes that ran give the selection.
- The retry of a tool call. A hook that sets `AfterToolCallEvent.retry` makes Strands run the tool
  again inside the same `execute_tool` span, which ends OK. Only the hooks see the first attempt, its
  exception type (`TimeoutError`), and the retry.
- The interrupt and the resume: `AfterInvocationEvent` gives the stop reason `interrupt` and the
  interrupt id, which contains the held tool call id; the person's response reaches only the hook
  that raised the interrupt.
- The sampling settings of each agent: `model.get_config()` gives the temperature and the output token
  limit, which the chat spans leave out. The app logs them once per agent.
- Every hook runs inside the span it belongs to, so the app logs the current span id with each event
  and the record joins the two paths by id.

## Declared by the application through OpenTelemetry spans

Strands records these only when the application opens its own span for them:

- each evaluator verdict and its reason (the evaluator loop is application code);
- the target of the effect.

## Neither path

- The cost of a model call. The OpenAI provider keeps only token counts from the usage block.
- The memory record a search returned or an add wrote: entries have no id in the spans, so the
  record matches them by content (knownBy inferred).
- Which earlier answer a node's or model's input came from. The record links an input to an earlier
  answer that appears in it word for word (knownBy inferred).
- A reliable parent of a loop turn. `event_loop.parent_cycle_id` points to cycles of other agents
  when agents run concurrently in a graph or one after another: in this run four of the six turns that
name a parent cycle name a cycle of another agent. The record orders turns by time within each agent
run instead.
- The time a memory record's content is about.
