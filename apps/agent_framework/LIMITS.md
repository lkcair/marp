# Microsoft Agent Framework: what each path delivers

Checked against run `maf-20261002T080958` (agent-framework-core 1.19.0, agent-framework-openai 1.14.4,
agent-framework-orchestrations 1.2.0, openai 3.23.0, opentelemetry-sdk 1.45.0; model
nvidia/nemotron-3.5-lightning:free through the OpenAI chat completion client pointed at OpenRouter).

The app runs three workflows in turn. `route` is a `WorkflowBuilder` graph: a coordinator executor
asks its agent which agents it needs and sends the task through a multi-selection edge group to the
weather and facts workers, which run in the same superstep and meet in a fan-in edge group. `handoff`
is a `HandoffBuilder` workflow in which the coordinator agent hands the work to the writer with the
handoff tool the builder injects. `review` is a `WorkflowBuilder` loop between an evaluator executor
and a reviser, ending in an `AgentExecutor` whose `send_brief` tool has
`approval_mode="always_require"`. The person's approval is a response to the workflow's
`request_info` event, sent with `workflow.run(responses=...)`.

## Spans (built-in OpenTelemetry instrumentation)

The framework traces through the global tracer provider, so the spans went to the local JSONL
exporter and nowhere else. `enable_instrumentation(enable_sensitive_data=True)` turns on message
content, and `OTEL_SEMCONV_STABILITY_OPT_IN=gen_ai_latest_experimental` selects the message
attributes of the current GenAI conventions.

- One `workflow.build` span per workflow, with `workflow.definition`: the executors with their
  classes, and every edge group with its type, its edges, and, for a multi-selection group, the name
  of the selection function (`<lambda>`). The candidates of the routing come from this definition.
- One `workflow.run` span per run, with the workflow name and id. Each run, and each build, is its
  own trace: the run has seven traces. `workflow.id` joins each build to its runs, and nothing joins the
  three workflows.
- One `executor.process` span per message an executor handled, with the executor id and class. The
  span of the executor that received the person's answer has message type `MessageType.RESPONSE`.
- One `edge_group.process` span per delivery, with the group type, the source, whether the message
  was delivered, and the target only for single and internal edges. Fan-out and fan-in deliveries
  name no target.
- One `message.send` span per message an executor sent, with the message class and, when the
  executor named one, the destination executor.
- One `invoke_agent` span per agent run, with the agent name and id, the instructions, the tool
  definitions, the request temperature and max tokens, the input and output messages, and token
  counts. Its provider is `microsoft.agent_framework`.
- One `chat` span per model call, with the request and response model, the response id, finish
  reasons, input, output, and reasoning tokens, and the input and output messages. The messages carry
  the id of every tool call the model requested and of every tool result it read. The provider is
  `openai`, and `server.address` names OpenRouter. The chat spans carry no temperature or max tokens;
  the converter takes them from the agent span.
- One `execute_tool` span per tool call that ran, with the tool name, description, type, call id,
  arguments, result, and duration. The failed call has status ERROR, `error.type` `TimeoutError`, and
  an exception event.
- The retried weather call is a second `execute_tool` span with a new call id, and nothing links it
  to the failed one.

## Only the workflow events and middleware

- The superstep of each executor run (`superstep_started` with its iteration), which shows that the
  weather and facts workers ran in the same superstep.
- The handoff, as a `handoff_sent` event with the source and target agents. The handoff tool the
  builder injects ends in `MiddlewareTermination`: the function middleware sees the call, and no
  `execute_tool` span is opened for it.
- The tool call held for approval: a `request_info` event with its request id (its data, the
  function approval request, is not serialized in this run; the app logs the call id and the tool). The held call has no `execute_tool` span;
  the call that runs after the approval has one, with the same call id, in a new `workflow.run`.
- The exception type and message of a failed tool call, from the function middleware. The function
  middleware gets no call id, so its events join the tool spans by tool name and order.
- Which agent each executor runs, read from the executor objects. The definition in the spans gives
  executor classes only, so it cannot name the agent behind the translator, which did not run.
- The workflow events carry no timestamp. The app ran the workflows without streaming and logged the
  events after each run, so their order holds and their times do not.

## Declared by the application

The framework records these only when the application reports them; the app writes them to its
event log with the id of the span they happened in, so they join the spans exactly:

- the stated reason of the routing (the coordinator agent's answer). The selection itself is a
  native fact: the workers that processed a message from the multi-selection group;
- the write and reads of the `notes` memory (a module-level dictionary, the facts executor writes
  it, the writer's `read_note` tool reads it);
- each evaluator verdict and its reason;
- the approval decision of the person;
- the target of the effect.

## Neither path

- The cost of each model call. OpenRouter returns it in the usage block; the chat completion client
  keeps the token counts in `usage_details`, drops the cost, and keeps no raw response, so neither
  the spans nor a chat middleware sees it.
- A conversation or session id. No span carries `gen_ai.conversation.id`; the framework sets it only
  from an internal context variable that the public API does not expose.
- What one agent passed to another. A workflow message's payload is in no span; the record links a
  model input to an earlier answer of another agent turn when the answer appears in the input word
  for word (knownBy inferred).
- What a memory record and a tool result came from, also matched by content (knownBy inferred).
- The time a memory record's content is about.

The framework also logs `Ignored an approval response ... did not match the active approval
occurrence identity` when the review workflow resumes, and then runs the approved call once.
