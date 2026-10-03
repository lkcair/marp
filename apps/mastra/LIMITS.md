# Mastra: what each path delivers

Checked against run `ma-20261002T092340` (@mastra/core 1.74.0, @mastra/memory 1.35.0,
@mastra/observability 1.18.3, @mastra/otel-exporter 1.4.4, @openrouter/ai-sdk-provider 3.1.0, Node
22.22; model nvidia/nemotron-3.5-lightning:free).

The app is a Mastra workflow in JavaScript. A coordinator step asks the coordinator agent which agents
to use, and `.branch()` runs every branch whose condition holds, so the weather and facts steps run
concurrently. In the handoff step the coordinator, a supervisor agent with the writer as its subagent,
delegates the draft to the writer through the `agent-writer` tool. A `.dountil()` loop runs the
evaluator and the writer's revisions, and the approve step suspends the workflow until the app resumes
it with the person's decision. The facts step writes the note to Mastra Memory, as working memory scoped
to the resource `notes`, kept in an in-memory store.

Mastra reports feature usage to PostHog unless `MASTRA_TELEMETRY_DISABLED` is set, and the app sets it.
The OpenTelemetry exporter is given a file exporter in place of its network exporter, so no span left
the machine.

## Spans (Mastra's OpenTelemetry exporter)

The exporter turns each span of Mastra's own tracing into one OpenTelemetry span with the same span id.
The 90 spans of the run share one trace: the resumed workflow run is a child of the suspended one.

- One `invoke_workflow` span per workflow run, with its status (`suspended`, then `success`), and one
  `workflow_step` span per step run, with its status. The approve step has a suspended span and a
  successful one.
- The branch as a `workflow_conditional` span with the number of conditions, the indexes that held, and
  the ids of the selected steps, plus one `workflow_conditional_eval` span per condition. The loop as a
  `workflow_loop` span with its type and its number of iterations.
- One `invoke_agent` span per agent run: agent id and name, system instructions, input, and output.
  It lists the names of the agent's tools (`gen_ai.tool.definitions`), but not which of them runs a
  subagent.
- One `chat` span per model call: request and response model, provider `openrouter`, temperature, output
  token limit, finish reason, and input, output, and reasoning tokens. It has no input messages, and its
  output messages hold one text part, so the tool calls a model requested do not appear in it. It has no
  response id and no cost.
- One `agent_step` span per step of an agent's loop and one `model_generation` span per call to the
  agent.
- One `execute_tool` span per tool call: tool name, description, call id, arguments, and result. The
  failed call has status ERROR, `error.type` `unknown`, and the error message. A delegation to a subagent
  is a tool call named `agent-writer`, with the subagent's run nested under it.
- A `memory_operation` span for the working memory update, with the thread and resource ids. It does not
  hold the written value.
- No `gen_ai.conversation.id`; only the memory update names a `threadId`, the run id.

## Only Mastra's own spans, callbacks, and objects

The app adds a second exporter to Mastra's observability that writes each span as Mastra holds it, and
logs each agent's `onStepFinish` callback.

- The input messages of each model step, and its output with the requested tool calls and their ids.
  The tool spans carry the same ids, so a model step links to the tool calls it requested. Tool results
  reach the next step's input only as the placeholder `[tool-result: <tool>]`; the record links the
  next model step of the same agent run to the results of the tools the previous step called.
- The error's name, `TimeoutError`, on the failed tool call.
- The cost. Each `model_generation` span has a cost context with the cost OpenRouter reported for the
  call to the agent, and `onStepFinish` gives OpenRouter's usage block, with the cost and the provider
  behind OpenRouter (Nvidia), for each step. The record matches the steps to the model calls by agent and
  token counts.
- From the agent objects, the tools of each agent and the subagents it can delegate to: the
  coordinator's supervisor can delegate to the writer.
- From the workflow object, its step graph: the steps of the branch, the source of each condition, the
  loop's type, and the step that can suspend. The steps of the branch, the translator included, are the
  candidates of the routing.
- The memory's scope, `resource`, from its configuration.
- The workflow's `watch` events: each step's start, result, finish, and suspension.

## Declared by the application through event spans

The app opens Mastra event spans for these, and Mastra exports their metadata as `mastra.metadata.*`
attributes:

- which agent each branch step runs, as step metadata, so that the translator, a candidate that did not
  run, has an agent;
- the written value of the note and each read of it (reads through `getWorkingMemory` make no span);
- each evaluator verdict and its reason (the loop body is application code);
- the person's decision, with which the app resumes the workflow;
- the target of the effect.

## Neither path

- A session: no span or event names a conversation beyond the `threadId` of the memory update.
- What a tool result or a memory record came from. The record links a memory record to the answer it
  equals, an input to an earlier answer that appears in it word for word, and a delegation's result to
  the subagent's answer it equals (knownBy inferred).
- The time a memory record's content is about.
