# Agno: what each path delivers

Checked against run `ag-20261002T092848` (agno 3.1.0, openai 3.23.0,
openinference-instrumentation-agno 1.0.13, opentelemetry-sdk 1.45.0; model
nvidia/nemotron-3.5-lightning:free).

Agno sends usage telemetry to its own service unless `AGNO_TELEMETRY=false` or `telemetry=False`; the
app sets both, so nothing left the machine except the model calls. Agno's own tracing setup wraps the
same OpenInference instrumentor and stores the spans in an Agno database; the app sends them to the
JSON lines exporter instead.

## Spans (OpenInference instrumentor)

- One AGENT span per agent or team run, named `<agent>.arun` (or `acontinue_run` for a resumed run),
  with the agent name, `agno.run.id`, and `session.id`.
- One LLM span per model call: model name, `llm.provider` `OpenRouter`, invocation parameters
  (temperature and max tokens), prompt and completion tokens, `llm.cost.total`, and the input and output
  messages. The output messages carry the id of every tool call the model requested, and the input
  messages carry the id of every tool result the model read.
- One TOOL span per tool call: tool name, description, parameter schema, arguments, and result. A failed
  call has status ERROR. The team's delegation to a member is a TOOL span named
  `delegate_task_to_member`, with the member and the task as arguments.
- One CHAIN span per workflow run, per executed step (`<step>.aexecute_stream`), and per parallel step.
  The router and the loop have no span of their own, and a loop's repeated steps carry no iteration
  number.
- The model and tool spans of an agent that a workflow step runs directly hang under the step's span,
  beside the agent's span (a team member's hang under its own agent span), so the spans alone do not say which agent made a call. Spans of one parallel
  branch can hang under the other branch's step.
- The spans of the workflow and of the agent runs outside it carry the workflow's session id; each
  agent run inside a step has a session id of its own, which a team member shares with its team. The workflow run and each top-level agent run
  are separate traces (three in this run).

## Only Agno's own events

The app reads the workflow, team, and agent run events from the event stream (`stream_events=True`).

- The workflow's structure: steps with their ids and parent step ids, the router with the steps it
  selected, the parallel step, and the loop with each iteration's number.
- Each agent and team run with its run id, the step it ran in, and the run that started it; the
  writer's run names the team run as its parent.
- The tool call id on every tool call, with its arguments, result, and error flag. A failed call is
  followed by a `ToolCallError` event with the message.
- The held tool call: the run pauses with a `RunPaused` event listing the call that needs confirmation;
  the call has no span until it runs. The executed call carries `confirmed: true`.
- Each agent's model settings, read from the agent objects.

The native model request events carry the tokens but no cost, and Agno's metrics drop every field
whose value is zero, so the cost of 0 that OpenRouter reports for free models reaches only the spans.

## Declared by the application

- Which agent runs in each step the router can choose, and the coordinator's stated reason, logged by
  the app next to the routing.
- The write of the `notes` note (a workflow custom event from the facts step) and each read of it (an
  agent custom event from the `read_note` tool, which Agno stamps with the id of the enclosing tool call; for a delegated member, the delegation's).
- Each evaluator verdict and its reason (a workflow custom event from the evaluate step).
- The target of the effect (an agent custom event from the `send_brief` tool).

The router's selection, the parallel run, the handoff, the loop, the retry, and the person's approval
are native.

## Neither path

- The router's selected agents as agents: the router selects steps, and the agents behind the steps
  are known from the runs that execute in them. A Router runs a list of selected steps one after
  another, so the app returns one Parallel step of them, which Agno runs with a warning that the step
  is not among the router's choices.
- The coordinator's routing call inside the selector: it runs outside the event stream, so only its
  spans record it.
- What a prompt was built from. A later prompt is linked to an earlier answer or delegated task whose
  text it contains (at least 16 characters, knownBy inferred), and the note to the facts answer by equal
  content.
- The exception type of a failed tool call: Agno reports the message only.
- The time of an event: Agno's events carry whole seconds, so the record uses the time each event
  reached the app.
- The time a memory record's content is about.
