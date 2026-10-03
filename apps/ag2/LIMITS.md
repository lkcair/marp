# AG2: what each path delivers

Checked against run `ag-20261002T155317` (ag2 1.1.1, openai 3.23.0, opentelemetry-sdk 1.45.0; model
google/gemma-4-26b-a4b-it on OpenRouter).

AG2 1.x is a rewrite: agents are `ag2.Agent` objects with middleware and observers, and the
`ConversableAgent` and `GroupChat` API of earlier versions is gone. The app delegates with
`subagent_tool`, which exposes an agent as a tool named `task_<agent>`. AG2 traces through its own
`TelemetryMiddleware`, which takes the tracer provider directly; the app gives every agent one, with
the run id stamped on every span as `gen_ai.conversation.id`, so no span left the machine.

## Spans (AG2's TelemetryMiddleware)

- One `invoke_agent` span per agent turn, with the agent name and the provider and model the app gives
  the middleware. The turn of an agent run as a delegation nests under the `execute_tool` span of that
  delegation, so the spans show which agent ran inside which call.
- One `chat` span per model call, with the request and response model, the finish reason, the tokens,
  and the input and output messages. The messages are in the OpenAI chat format (`tool_calls` with ids,
  `role: tool` with `tool_call_id`), not the GenAI parts format, so a reader of the GenAI conventions
  does not find the tool call ids in them. The span has no temperature, no output limit, no response id,
  and no cost.
- One `execute_tool` span per tool call: tool name, call id, arguments, and result. The failed call has
  status ERROR, the error message, and an `exception` event with the exception type, with no `error.type`.
- One `await_human_input` span for the person's approval, under the `send_brief` call it gates, with
  the prompt and the answer.
- One `record_usage` span per model call, with the tokens; its provider is `openai`, the client class.
- Each top-level `ask` is its own trace, six in this run; the conversation id joins them.

## Only the native observers and objects

The app gives every agent an observer that logs the events on its stream.

- The model responses with their tool calls and call ids, which join the chat spans of the same agent
  in order, and the tool calls, results, and errors with their call ids, which join the tool spans
  exactly.
- `TaskStarted` and `TaskCompleted` for each delegation, with the task id, the agent, and the objective.
- The person's answer as a `HumanMessage` naming the request it answers.
- From the agent objects, the tools of each agent; the `task_` tools of the coordinator are the
  candidates of its routing, the translator included. The delegations it called are the selection, and
  the delegation whose result the coordinator gives as its own final answer is the handoff.
- The approval itself: `approval_required()` on `send_brief` asks the person through the agent's
  `hitl_hook`, which receives a `ToolApprovalRequest` with the id of the call it gates. The app answers
  it on behalf of a person.
- The sampling settings, which the app logs from its model configuration by an explicit allowlist (the
  configuration also holds the API key).

## Declared by the application

- the write of the `notes` memory, by a middleware on the facts delegation, and each read, from the
  `read_note` tool;
- each evaluator verdict and its reason (the evaluator loop is application code);
- the person's decision, logged by the hook that answers the request;
- the target of the effect.

## Neither path

- The cost of a model call. AG2's usage keeps token counts only, although OpenRouter returned a cost.
- A response id that would join a chat span to its model response exactly; the record joins them by
  agent and order.
- The retry link. AG2 returns the error to the model, which calls the tool again; the record links the
  two calls because they have the same tool and the same arguments.
- What a model input came from. The record links an input to an earlier answer that appears in it word
  for word, and a delegation's result to the delegated agent's answer it equals (knownBy inferred).
- The time a memory record's content is about.

In this run the final brief left out the weather. The evaluator answered FAIL at iterations 2 and 3,
and the loop ended at its limit of three, which the app records as a pass.
