# CrewAI: what each path delivers

Checked against run `cr-20261002T075925` (crewai 1.15.23, openai 2.54.0, opentelemetry-sdk 1.45.0;
model nvidia/nemotron-3.5-lightning:free).

CrewAI sends anonymous usage telemetry to its own servers and can upload execution traces to its
platform. The app turns both off (`CREWAI_DISABLE_TELEMETRY`, `CREWAI_DISABLE_TRACKING`,
`CREWAI_TRACING_ENABLED=false`) and opens CrewAI's own trace session with our tracer provider, so the
spans go to `spans.jsonl` and nothing left the machine except the model calls.

## Spans (CrewAI's built-in tracing)

CrewAI turns the events of its event bus into OpenTelemetry spans under the GenAI conventions. Every
span carries `event_id`, and every span but the flow's root carries `parent_event_id`, the identifiers of the native event that started it, so
spans and native events join exactly.

- One `invoke_workflow` span for the flow, with its inputs, the names of its methods, and its id
  (`crewai.flow.id`), and one for each crew, with its process and token totals.
- One `execute_method` span per run of a flow method, with its parameters, result, and the flow state
  after it. The result of a router is the route it chose.
- One `invoke_agent` span per agent run. A standalone agent has its role as `gen_ai.agent.name`; an
  agent inside a crew has a hash there and its role only in `crewai.agent.role`.
- One `execute_task` span per task, with its description, expected output, and output.
- One `chat` span per model call: provider (`openrouter`), request and response model, temperature,
  max tokens, stop sequences, response id, finish reasons, input, output, and reasoning tokens, the
  input and output messages, and the tool definitions. A tool call in the output messages has its id.
- One `execute_tool` span per tool call, with the tool name, arguments, and result. The failed call
  has status ERROR and `error.type` TimeoutError, and a second `execute_tool` span named
  `tool failure` holds the error message.
- One span per memory write, with the value and metadata, and one per memory query, with the query
  and the records found, each with its id and scope.
- One span when human feedback is requested and one when it is received, with the feedback text.
- `gen_ai.conversation.id` holds the id of the agent that made the call, or of its task on some model
  calls, so it differs from agent to agent and names no conversation.

## Only the native event bus and the flow definition

- Which method's completion started each method (`triggered_by_event_id`), which gives the edges of
  the flow as they ran: the two research agents started from the router, the handoff from the last of them to finish,
  the evaluation from the handoff and from each revision.
- The routes each router can choose, from `flow_definition()` with `emit` on the router, and the
  methods that listen to each route, which give the candidates of a routing.
- Events of the application's own types, such as the evaluator's verdicts.

## Declared by the application

- Which agent each flow method runs, so that the translator, a candidate that did not run, has an agent.
- Each evaluator verdict and its reason, as an event of a type the application defines; CrewAI emits
  such events on its bus and makes no span for them.
- The person's decision. CrewAI emits the human feedback events from its console provider; a provider
  that answers for a person emits the same events itself, as the app's simulated person does.

## Neither path

- That a tool changed the world outside the run. The converter treats the result of a tool call made by
  a method that the person's approval started, through `@human_feedback`, as the effect, with that
  result as its target.
- The cost of a model call: the usage CrewAI keeps has tokens only.
- The id of a tool call on its tool span and in the next model input. The converter gives each tool
  call the model output that preceded it in the same agent loop, and gives the tool result to the next
  model call of that loop.
- What an agent passed to another through the flow state. The converter links a model input or tool
  arguments to an earlier answer, tool result, or memory record whose text they contain (at least 16
  characters), marked knownBy inferred.
- A way to force a tool call: CrewAI always sends `tool_choice` auto, so the app asks the sender again
  until the brief is sent.
- The time a memory record's content is about; the record has its creation and access times.
