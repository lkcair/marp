# AgentScope: what each path delivers

Checked against run `as-20261002T155628` (agentscope 2.0.9, openai 3.23.0, opentelemetry-sdk 1.45.0;
model google/gemma-4-26b-a4b-it through OpenRouter).

The app is an AgentScope 2 `TeamPipeline`: the coordinator is the leader, and the weather, facts,
translator, and writer agents are its members. The leader assigns work with the native `TeamAssign`
tool, and members assigned in the same round run concurrently, so the weather and facts agents ran in
parallel. The writer then received the work through a third assignment. The evaluator loop is
application code, and `send_brief` carries the permission "ask", so the writer's call is held until a
person confirms it. AgentScope's `TracingMiddleware` writes OpenTelemetry spans to the global tracer
provider, which the app sets to a local file, so no span left the machine.

## Spans (AgentScope's TracingMiddleware)

The 35 spans of the run share one trace, under the app's root span.

- One `invoke_agent` span per reply, with the agent name and `agentscope.agent.reply_id`. A reply that
  resumes after its members answered, or after a person confirmed a call, gets a new span with the same
  reply id and `agentscope.agent.incoming_event_type` (`external_execution_result` or
  `user_confirm_result`). The pending calls are named in `agentscope.agent.external_execution_pending_tools`
  and `agentscope.agent.hitl_pending_tools`. `gen_ai.agent.description` is "The agent class." on every
  agent.
- One `chat` span per model call: request model, provider `openai` (the client), response id, finish
  reasons, input, output, and cached tokens, and the input and output messages. The output messages
  carry the id and arguments of every tool call the model requested, including each `TeamAssign` with the
  member it names, and the input messages carry the id of every tool result the model read. No span has
  a temperature or an output limit.
- One `execute_tool` span per tool call: tool name, call id, arguments, and the result as the repr of a
  `ToolResponse`. A `TeamAssign` span has the member's answer as its result and no arguments, and is
  marked `agentscope.agent.is_external_execution`.
- The failed `get_weather` call has a span with status OK.
- `gen_ai.conversation.id` differs for each agent: the five agents of the run have five conversation
  ids, so the spans do not name one session for the run.

## Only the native event stream, middleware, and objects

- The reply events with the agent name and reply id, which join the agent spans exactly.
- The state of each tool result: the first `get_weather` call ended with state `error`.
- The held call: `REQUIRE_USER_CONFIRM` names the `send_brief` call with its id and arguments. It has no
  span of its own until it runs after the confirmation.
- A middleware on every agent logs the response id of each model call and the current span id, which
  join the native model calls to the chat spans exactly.
- From the objects: the members of the team, which are the candidates of the routing, and the tools of
  each agent.

## Declared by the application

- The write of the `notes` memory (a middleware on the facts agent writes its answer) and each read of
  it inside `read_note`, with the span id of the current span.
- Each evaluator verdict and its reason (the loop is application code).
- The person's decision: the app answers the confirmation request and logs the decision.
- The target of the effect, logged inside `send_brief`.
- The sampling settings, from the app's own configuration.

## Neither path

- The cost of a model call: AgentScope's usage keeps tokens and time only.
- The exception type of a failed tool call: the toolkit turns the exception into an error result with
  its message.
- A session for the run: the app's root span carries `session.id`, which the app sets.
- What a tool result came from. The result of a `TeamAssign` is linked to the member's last answer, and
  a model input to an earlier answer that appears in it word for word, by content (knownBy inferred).
- The time a memory record's content is about.
