# Pydantic AI: what each path delivers

Checked against run `pa-20261002T065003` (pydantic-ai-slim 2.53.0, pydantic-graph 2.53.0, openai 3.23.0,
opentelemetry-sdk 1.45.0; model nvidia/nemotron-3.5-lightning:free through Pydantic AI's OpenRouter model).

Pydantic AI has its own OpenTelemetry instrumentation, turned on with `Agent.instrument_all` and sent to
a local file. The app uses data format version 6, the one whose tool results carry the GenAI role
`tool`. Nothing is sent to Logfire.

## Spans (Pydantic AI instrumentation)

- One `invoke_agent` span per agent run, with the agent name, the run id (`gen_ai.agent.call.id`), the
  final result, the aggregated tokens of the run, and all its messages.
- One `chat` span per model call: request and response model, provider `openrouter`, server address
  `openrouter.ai`, temperature, max tokens, response id, finish reasons, input, output, and reasoning
  tokens, the system instructions, the tool definitions, and the input and output messages.
- One `execute_tool` span per tool call: tool name, call id, arguments, and result. A tool that asks
  the model to try again has status ERROR and an exception event of type `ToolRetryError` with the
  message.
- The output function that hands work to the writer appears as `execute_tool final_result`, with the
  arguments the coordinator passed.
- `gen_ai.conversation.id` on every span. Each top-level run is its own trace; the six traces of the
  task share only the conversation id.
- An agent called from a tool runs inside that tool's span, so the spans nest the worker runs, and the
  writer's run, under the coordinator.
- The tool call id appears in the model output that requested the call, on the tool span, and in the
  next model input that read the result.

## Only the native API

The app records the native data with a capability attached to every agent: its run hooks, its model
request hook, and its view of the run's event stream.

- The step of each model call within its run (`run_step`), which numbers the agent's loop.
- That a tool result was a request to retry (`RetryPromptPart`), and so which later call of the same
  tool is the retry.
- The tool call held for approval (`DeferredToolRequestsEvent`). A held call is not executed, so it
  has no span; the call approved later runs in a new run with the same call id.
- The service OpenRouter routed each call to (`downstream_provider`, Nvidia here).
- The application's custom events (`CustomEvent` subclasses emitted with `RunContext.emit` or
  `AgentRun.emit`). Pydantic AI stamps each event emitted inside a tool with the tool call id, which
  joins it to the tool span exactly.

## Declared by the application through custom events

- which of the coordinator's tools run which agent, and that its output function hands off to the
  writer (Pydantic AI delegates to an agent through an ordinary tool and has no handoff primitive);
- the handoff from the coordinator to the writer;
- reads and writes of the `notes` memory, with the written value;
- each evaluator verdict and its reason (the evaluator emits it from its output validator; the loop is
  application code, so the spans show separate runs and no loop);
- the approval decision of the person;
- the target of the effect.

## Neither path

- The cost of a free model call. OpenRouter returns a cost of 0, and Pydantic AI copies the cost into
  `provider_details` only when it is not zero.
- The exception the tool first raised. The app turns the `TimeoutError` into `ModelRetry`, as Pydantic
  AI expects, and only the message survives.
- What a tool result came from. The result of a tool that runs an agent is linked to that agent's final
  answer, and the note to the facts agent's answer, because their contents are equal (knownBy
  inferred).
- The time a memory record's content is about.
