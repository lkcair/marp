# Haystack: what each path delivers

Checked against run `hs-20261002T155430` (haystack-ai 3.3.0, openai 3.23.0,
openinference-instrumentation-haystack 0.1.44, opentelemetry-sdk 1.45.0; model
google/gemma-4-26b-a4b-it through OpenRouter).

The app is one Haystack pipeline run with `run_async`, which runs the components that are ready at the
same time concurrently. A custom coordinator component asks the model which agents to use and emits a
task on the output of each selected agent, so the weather and facts `Agent` components run in parallel.
A handoff component joins their answers and passes the work to the writer. A `ConditionalRouter` sends a
failed draft back to the writer through a `BranchJoiner` and a passed draft to the sender, an `Agent`
whose `send_brief` tool is guarded by a `ConfirmationHook`; the sender has parallel tool calls off and
stops once `send_brief` has run, so the brief is sent once. The note lives in an `InMemoryDocumentStore`.

Haystack 3.3 has no OpenTelemetry tracer of its own: its tracing module defines a `Tracer` interface and
a logging tracer. The spans come from the OpenInference instrumentor, written to a local file, and
Haystack telemetry is off (`HAYSTACK_TELEMETRY_ENABLED=False`), so no span left the machine.

## Spans (OpenInference instrumentor)

- One CHAIN span for the pipeline run (`Pipeline.run_async`) and one for its generator
  (`Pipeline.run_async_generator`).
- One CHAIN span per run of a custom component (`Coordinator.run`, `Handoff.run`, `Evaluator.run`) and of
  the routers and joiners (`ConditionalRouter.run`, `BranchJoiner.run`).
- One CHAIN span per agent run (`Agent.run_async`). No span has the AGENT kind and no attribute names an
  agent or the component it runs in.
- One CHAIN span per prompt the agent builds (`ChatPromptBuilder.run`).
- One LLM span per model call: model name, `llm.provider` (`openai`, the client; OpenRouter is not
  named), prompt and completion tokens, and the input and output messages. The output messages hold the
  name and arguments of each tool call the model requested, without its id. There are no invocation
  parameters, so no temperature or output limit. No structured attribute holds a call id or a cost; the
  raw `output.value` holds both.
- No tool spans: the tools run inside the agent's tool invoker, which the instrumentor does not wrap.
- `session.id`, because the app runs the pipeline inside `using_session`.

## Only Haystack's own tracer

The app implements Haystack's `Tracer` interface and logs every span it opens, with its tags, its parent,
and the OpenTelemetry span current when it opened. That span id joins the two paths exactly: an
`Agent.run_async` span has the id the native agent run recorded, and the model calls of a step are the
LLM spans under it, in order. With `HAYSTACK_CONTENT_TRACING_ENABLED=true` the tags hold the content.

- One `haystack.component.run` span per run of a component: its name, type, number of visits (the
  iteration of a loop), inputs and outputs, and the senders and receivers of each socket. The outputs of
  the coordinator give the agents it selected; its connections give the candidates, the translator
  included.
- `haystack.agent.run`, `haystack.agent.step` with the step number, and `haystack.agent.step.llm` with
  the full input and output messages: the id of every tool call the model requested, the id of every tool
  result it read, and the usage block OpenRouter returned, cost included.
- `haystack.agent.step.tool` with the tool name, arguments, and output. A failed tool returns an error
  dictionary as its output, and the span ends without an error.
- `haystack.agent.hook` for each hook run, and `haystack.agent.hook.human_in_the_loop.strategy` with the
  tool call id and the decision (`confirm`) for the call a person reviewed.

## Only the pipeline object

- The components, the type of each, the connections between their sockets, and the tools of each agent.

## Declared by the application through Haystack's own tracer

The app opens its own spans for these through `tracing.tracer.trace`:

- the coordinator's stated reason for its routing. The application keeps only the weather and facts
  agents from the coordinator's choice, so the selection is the application's;
- the handoff from the coordinator to the writer (the handoff component is application code);
- the write and reads of the `notes` memory, with the written value;
- each evaluator verdict and its reason (the evaluator is a custom component);
- the target of the effect.

The app also logs the sampling settings it built each chat generator with, since no span carries them.
The converter takes the agent of a component from the agent named in the facts declared inside it, so the
sender component acts for the writer, which declares the effect from it.

## Neither path

- The tool call id on a tool span. The converter pairs each tool span with the call of the same name and
  arguments that the model requested in the same step.
- The exception type of a failed tool call: the error dictionary holds the message only.
- The retry of a failed tool call. The tool returns the error to the model, which calls the tool again;
  the record links the two calls because they have the same tool and the same arguments.
- The cost of the model calls the coordinator and the evaluator make themselves: these run outside an
  agent, so only their LLM spans record them, with the cost only in the raw `output.value`.
- What a component's output came from. The output of an agent component is linked to the agent's final
  answer, and the note to the facts agent's answer, because their contents are equal (knownBy inferred).
- The time a memory record's content is about.
