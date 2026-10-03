# LlamaIndex: what each path delivers

Checked against run `li-20261002T083059` (llama-index-core 0.14.25, llama-index-workflows 2.25.0,
llama-index-llms-openai-like 0.8.1, openai 2.54.0, openinference-instrumentation-llama-index 4.5.4,
opentelemetry-sdk 1.45.0; model nvidia/nemotron-3.5-lightning:free).

The app is a LlamaIndex Workflow. A coordinator step sends one event to each selected worker step,
the weather and facts steps run concurrently, and a merge step collects both events. The handoff
runs inside the merge step as an `AgentWorkflow` in which the coordinator agent calls the built-in
`handoff` tool to pass the work to the writer. The evaluator loop is a cycle of workflow events, and
the approval is an `InputRequiredEvent` answered with a `HumanResponseEvent`. The weather step has a
retry policy: when its tool fails, the step raises and the workflow runs it again.

## Spans (OpenInference instrumentor)

The OpenInference instrumentor turns every span of LlamaIndex's instrumentation dispatcher into one
OpenTelemetry span, in the same order.

- Every span carries `session.id`, set by the app with `using_session`.
- Workflow steps are CHAIN spans named by the step function's qualified name
  (`build.<locals>.CityBrief.weather`). Their input and output are the reprs of the workflow events,
  and the output repr cuts the text of an event after about 50 characters; the input repr is complete.
- No span has the AGENT kind and no attribute names an agent: an agent run is a CHAIN span named
  `FunctionAgent.run`, and the turns of an `AgentWorkflow` are CHAIN spans named `run_agent_step`.
- Each model call appears as two nested `OpenAILike.achat` LLM spans, and a call with tools adds a
  third LLM span, `OpenAILike._prepare_chat_with_tools`. Only the inner `achat` span has the token
  counts and the input and output messages: in the run, 45 LLM spans stand for 16 model calls.
- The output messages carry the id of every tool call the model requested, and the input messages
  carry the id of every tool result the model read.
- `llm.invocation_parameters` holds the model metadata (context window, output limit, model name)
  and no temperature. `llm.provider` and `llm.system` are `openai`, the client class; OpenRouter is not
  named.
- One TOOL span per tool call, `FunctionTool.acall`, with the tool name, description, parameter
  schema, arguments, and result. The handoff is a TOOL span of the `handoff` tool, with `to_agent` and
  the reason in its arguments.
- A failed tool call and a failed step have status ERROR and an `exception` event with the type
  (`TimeoutError`). The retried step is a second span with no link to the first.

## Only the native instrumentation and workflow events

The app adds a span handler and an event handler to LlamaIndex's root dispatcher and reads the
workflow's event stream with `expose_internal=True`.

- The span handler sees the same spans with LlamaIndex's own span ids and parent ids, and the
  instance behind each span, which gives the name of each agent and tool.
- `LLMChatEndEvent` carries the raw usage OpenRouter returned, cost included, and is stamped with
  the span id of the inner `achat` span, which picks the model calls out of the nested LLM spans.
- `StepStateChanged` gives every run of every step, with the event type that triggered it and the
  type it returned. Together with the event values on the step spans, this gives the data flow
  between steps: which step produced the event another step consumed, the two events the merge step
  collected, and the loop of drafts and revisions.
- A failed run of the weather step followed by a run of the same step on the same event is the retry
  of the step's retry policy.
- The `AgentWorkflow` stream names the current agent of each turn (`AgentInput`) and gives each tool
  call with its id (`ToolCall`), so the handoff has the agent that handed off and the agent that
  received the work.
- `InputRequiredEvent` is the request for approval, and the step run triggered by
  `HumanResponseEvent` is the send that followed it.

## Only the framework objects

- The steps of the workflow, with the events each accepts and returns and whether it has a retry
  policy. The events the coordinator step can return give the candidates of the routing.
- Each agent's tools and the agents it can hand off to.
- The temperature, the output limit, and the API base, `https://openrouter.ai/api/v1`, of the LLM
  object.

## Declared by the application through dispatcher events

LlamaIndex records these only when the application dispatches them as instrumentation events:

- the write and reads of the `notes` memory, kept in the workflow's `ctx.store`, with the written value;
- each evaluator verdict and its reason;
- the target of the effect.

The person's answer to the approval request is sent by the application, which logs it.

## Neither path

- The tool call id on the tool spans. The converter pairs each tool span with the requested call of
  the same name in the same agent run.
- What a workflow event came from. The event that a step returns is linked to the final answer of
  the agent that ran inside the step when the visible text of the event starts that answer (knownBy
  inferred).
- The agent behind a workflow step. The converter takes the one agent that runs inside the step, else
  the agent that declared facts from it, else the step itself, so the translator step, which has no
  agent and did not run, is a candidate under its step name.
- A notion of scope for `ctx.store`: it lives with the workflow's context.
- The time a memory record's content is about.
