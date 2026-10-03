# LangGraph: what each path delivers

Checked against run `lg-20261002T103740` (langgraph 1.2.12, langchain 1.4.3, langchain-openai 1.6.7,
opentelemetry-instrumentation-genai-langchain 1.2b0, opentelemetry-sdk 1.45.0; model
nvidia/nemotron-3.5-lightning:free).

## Spans (official OpenTelemetry GenAI instrumentor)

Content capture is off by default and was turned on with
`OTEL_INSTRUMENTATION_GENAI_CAPTURE_MESSAGE_CONTENT=SPAN_ONLY`.

- One `invoke_workflow` span per graph invocation, so a run interrupted for approval shows two.
- One `invoke_agent` span per run of an agent made with `create_agent`, named by the agent; a failed
  run has status ERROR and `error.type`.
- One `chat` span per model call: request model, response model, provider, temperature, max tokens,
  finish reasons, input, output, and reasoning tokens, and the input and output messages. The
  provider is `openai`, the client class; no attribute names OpenRouter, the service that answered.
- One `execute_tool` span per tool call: tool name, description, arguments, and `error.type` when it
  failed, with the call id and result when the model requested the call (the node calls `send_brief`
  directly, so its span has neither).
- `gen_ai.conversation.id`, equal to the LangGraph `thread_id`.
- The tool call id appears in the model output that requested the call and in the next model input
  that read its result, so model calls and tool calls link by id.

## Only the native callbacks

- Every graph node as its own step, with its parent, the superstep (`langgraph_step`), the triggers,
  and the checkpoint namespace. The spans have no node spans: model calls made inside a node hang
  directly under the workflow span.
- The state keys each node read and wrote, which give the data flow between nodes.
- The retry of a node: LangGraph runs the node again in the same superstep after the failed attempt.
  The spans show two agent spans with no link between them.
- The loop: how many times the writer and the evaluator ran.
- The interrupt and the resume for the human approval. The instrumentor has no handler for these
  callbacks and logs `AttributeError` for `on_interrupt` and `on_resume`.
- The cost of each model call, which OpenRouter returns in the usage block of the response.

## Declared by the application through custom events

LangGraph records these only when the application reports them with `dispatch_custom_event`, so the
record holds them as the application states them:

- the routing decision: candidates, selected agents, and the stated reason;
- the handoff from the coordinator to the writer;
- reads and writes of the `notes` store, with the written value;
- each evaluator verdict and its reason;
- the approval decision of the person;
- the target of the effect.

## Neither path

- The service that answered the model calls, OpenRouter. The record names the client's provider.
- The value a node returned, beyond the names of the state keys it wrote.
- Agents created during the run: LangGraph has no such construct.
- The time a memory record's content is about.
