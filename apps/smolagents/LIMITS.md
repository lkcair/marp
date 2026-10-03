# smolagents: what each path delivers

Checked against run `sm-20261002T155326` (smolagents 1.26.0, openai 3.23.0,
openinference-instrumentation-smolagents 0.1.42, opentelemetry-sdk 1.45.0; model
google/gemma-4-26b-a4b-it through OpenRouter).

The app is a coordinator `ToolCallingAgent` whose managed agents are the weather, facts, translator,
and writer agents. smolagents runs the tool calls of one model step in a thread pool, so the weather and
facts agents, called in the same step, ran at the same time. The writer's answer goes through a
`final_answer_checks` function that asks an evaluator agent; a failed check makes the writer revise.

The model answered poorly in places: the coordinator first called the writer without the city, that
writer run ended on the forced pass of the third check, and the coordinator called the writer again,
whose draft passed the fourth check. The record holds these runs as they happened.

## Spans (OpenInference instrumentor)

- One AGENT span per agent run, named `<agent>.run`, with the task, the maximum number of steps, and the
  names of the agent's tools. The coordinator's span lists only `final_answer`: its managed agents are
  not among the tool names.
- One CHAIN span per step, named `Step <n>`. A step that failed with a parsing or tool error has status
  ERROR with the error class and message.
- One LLM span per model call, named `OpenAIModel.generate`: model name, temperature and output limit in
  the invocation parameters, prompt and completion tokens, input and output messages, and the id of every
  tool call the model requested. `llm.provider` is `openai` and `llm.system` is `vertexai`; no attribute
  names OpenRouter. No attribute holds a cost; the raw `output.value` of each model span holds the usage with its cost.
- One TOOL span per tool call, named after the tool class (`SimpleTool`, `FinalAnswerTool`), with the
  tool name, description, parameters, arguments, and result. The failed `get_weather` call has status
  ERROR with `TimeoutError: weather service timed out`. Tool spans carry no call id, and the next model
  input has the result only as text.
- A call to a managed agent has no TOOL span: the agent's AGENT span nests directly under the caller's
  step.
- No span carries a session or conversation id.

## Only the agents' memory, callbacks, and objects

The app registers step callbacks on every agent and logs each step as smolagents keeps it in the agent's
memory.

- Each action step with its step number, start and end time, the tool calls with their ids, the
  observations, the error class and message, and the tokens. The start time is within a few milliseconds of the start of the
  step's span, which joins the two paths.
- The raw usage OpenRouter returned for each call, with its cost, as a structured field.
- A failed check: the check function raises, and the step ends with `AgentError` ("Check review failed
  ...") in memory, while its span has status OK.
- The step an agent takes when it reaches `max_steps`: smolagents builds it outside the step wrapper, so
  it has no span (`_handle_max_steps_reached`); the record builds that step
  from memory.
- Each agent's final answer. The answer is the argument of the `final_answer` call that the run's last
  model call made, so the record derives it from that model output.
- From the agent objects: each agent's tools and managed agents, which give the candidates of the
  coordinator's routing, and the checks on its final answer.

## Declared by the application

- The handoff: smolagents has no handoff, so the app declares that the coordinator's call of the writer
  passes the work on.
- The write of the note to the shared `notes` memory, from a callback on the facts agent's final answer,
  and each read of it in the writer's `read_note` tool.
- Each evaluator verdict and its reason, from the check function.
- The approval of the person, given inside `send_brief` before the file is written.
- The target of the effect.

## Neither path

- A session: smolagents names no conversation.
- The tool call id of a tool span. The converter pairs each tool span with the model's requested call of
  the same name in the same step.
- The retry of a failed tool call. The model calls the tool again in the next step; the record links the
  two calls because they have the same tool and the same arguments.
- What a memory record and a tool result came from. They are linked to the answer whose content they
  equal (knownBy inferred).
- The service that answered the model calls, OpenRouter.
- The time a memory record's content is about.
