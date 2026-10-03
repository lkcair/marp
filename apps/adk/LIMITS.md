# Google ADK: what each path delivers

Checked against run `adk-20261002T070543` (google-adk 2.11.0, litellm 1.103.2, google-genai 2.27.0,
opentelemetry-sdk 1.42.1; model nvidia/nemotron-3.5-lightning:free through LiteLLM's OpenRouter
route).

ADK 2.11 marks `SequentialAgent`, `ParallelAgent`, and `LoopAgent` deprecated in favor of `Workflow`,
and a `Workflow` cannot be an `LlmAgent` sub-agent, so it cannot receive a transfer. The app uses
the agent features that still carry a handoff: an `LlmAgent` coordinator that calls the workers as
`AgentTool`s and transfers to the writer with `transfer_to_agent`. The workers run in parallel
because ADK runs the function calls of one model response concurrently.

## Spans (ADK's built-in OpenTelemetry tracing)

ADK traces through the global tracer provider, so the spans went to the local JSONL exporter and
nowhere else.

- One `invocation` span per runner call, including the inner runner of each agent used as a tool.
- One `invoke_agent` span per agent run, with the agent name, its description, and
  `gen_ai.conversation.id`, the session id. An agent used as a tool runs in its own session, so its
  conversation id differs from the caller's.
- One `call_llm` span per model call, with ADK's event id, invocation id, and session id, the
  request model, max tokens, finish reasons, token counts (input, output, cached, reasoning), and
  the full request and response as ADK's own JSON attributes. Its child `generate_content` span
  carries the OpenTelemetry GenAI attributes: input and output messages, system instructions, tool
  definitions, and token counts. The messages carry the id of every tool call the model requested
  and of every tool result it read.
- The temperature appears only inside ADK's request attribute, `gcp.vertex.agent.llm_request`.
- `gen_ai.system` is `gcp.vertex.agent` and no attribute names the provider; the model name carries
  the `openrouter/` route.
- One `execute_tool` span per tool call, with the tool name, description, type, call id (the same id
  the model gave the call), arguments, and response. The transfer to the writer is an `execute_tool`
  span named `transfer_to_agent`.
- When the model asks for several tools at once, an extra `execute_tool (merged)` span records the
  merged response. The converter skips it.
- A failed tool call handled by `ReflectAndRetryToolPlugin` has status UNSET and no `error.type`: the
  plugin turns the exception into a response, and the span records that response.
- The call held for approval is an `execute_tool` span whose response is
  `This tool call requires confirmation, please approve or reject.`

## Only ADK's events and plugin callbacks

- The exception type of a failed tool call (`TimeoutError`), from `on_tool_error_callback`.
- The retry count, from the response of `ReflectAndRetryToolPlugin` (`retry_count`).
- The transfer target in `actions.transfer_to_agent`.
- The confirmation request (`adk_request_confirmation`, carrying the original call) and the
  person's answer. The answer reaches the plugin through `on_user_message_callback`; it is not a
  runner event and no span records it.
- State changes in `actions.state_delta`, with the agent that made them. An agent used as a tool
  runs in its own session; `AgentTool` forwards its state changes to the caller, so they appear
  twice, once from the agent and once from the caller.
- The scope of a state key, from its prefix: `app:` is shared by every user and session of the
  application, `user:` by the sessions of one user, and `temp:` lasts one invocation. The note's
  record carries the scope `app`.

## Only the agent objects

- Which tools are agents (`AgentTool`) and which agents can receive a transfer (`sub_agents`), which
  give the candidates of a routing.
- Which tools require confirmation.

## Only LiteLLM

- The cost of each model call. OpenRouter returns it in the usage block; ADK's `usage_metadata`
  keeps the token counts and drops the cost. A LiteLLM `CustomLogger` sees the raw usage. The
  converter joins it to ADK's model calls by the tool call ids the completion returned, or by its
  answer text.

## Declared by the application through state changes

ADK records these only when the application writes them to session state:

- the write and reads of the `notes` memory. The note is app-scoped state (`app:notes/Lisbon`),
  which outlives the session; the facts agent's `after_agent_callback` writes it;
- each evaluator verdict and its reason (the evaluator loop is application code);
- the target of the effect.

## Neither path

- The ids of other agents' tool calls as the writer sees them. ADK rewrites the coordinator's calls
  into quoted text (`[coordinator] \`weather_agent\` tool returned result: ...`) before the writer's
  model call. The converter links the writer's call to the quoted result by agent, tool name, and
  time.
- What a tool result came from. The result of an agent used as a tool is linked to that agent's
  final answer, and the note to the facts agent's answer, because their contents are equal (knownBy
  inferred).
- The model's reasoning and its answer arrive as two text parts of the same message; the answer is
  the last one.
- The branch of a parallel run: the workers ran as concurrent tool calls, so every event has branch
  None.
- The time a memory record's content is about.
