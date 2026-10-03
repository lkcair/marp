# DSPy: what each path delivers

Checked against run `ds-20261002T115035` (dspy 3.4.0, litellm 1.103.2, openinference-instrumentation-dspy
0.1.47, opentelemetry-sdk 1.45.0; model cohere/north-mini-code:free through OpenRouter).

DSPy builds programs from modules. The app writes each agent as a module with an `agent_name`: the
coordinator is a `Predict`, the weather and facts agents are `ReAct` modules with their tools, the writer
and the evaluator are `Predict` modules, and a small release module holds the `send_brief` tool. The two
workers run as concurrent `acall`s. Each agent has its own `dspy.LM`, so the usage of a call can be read
from that LM's history. The OpenInference instrumentor sends its spans to a local file, so no span left
the machine.

## Spans (OpenInference instrumentor)

- One LLM span per model call, named `LM.acall`: model name, provider `openrouter`, invocation
  parameters (temperature and output limit), the input messages, and the raw completion as
  `output.value`, reasoning included. No span has token counts or output messages.
- One CHAIN span per module call, named after the class (`Coordinator.aforward`, `Worker.aforward`,
  `Facts.aforward`, `Writer.aforward`, `Evaluator.aforward`, `Release.aforward`, `ReAct.aforward`,
  `ChainOfThought.aforward`, `Predict.aforward`). Each `Predict` call adds two more CHAIN spans, its inner
  `Predict(StringSignature).forward` and the adapter's `ChatAdapter.acall`. No span has the AGENT kind and
  no attribute names an agent.
- One TOOL span per tool call, named `<tool>.acall`, with the arguments and the result as input and output
  values. The failed `get_weather` call has status ERROR and the description `TimeoutError: weather service
  timed out`. Tool spans have no `tool.name` attribute and no call id.
- Each top-level module call is its own trace: the run has 7 traces, joined only by `session.id`, which
  the app sets with `using_session`.
- No cost and no token counts.

## Only DSPy's callbacks and objects

The app registers a `BaseCallback` for every module, model, and tool call.

- The id of each call and the id of the call it ran inside. DSPy marks a call as active only after its
  start handler runs, so the active call seen there is the parent. The ids give the tree of modules,
  model calls, and tool calls, and the class and `agent_name` of each module.
- The turns of each `ReAct` loop, as the order of its `Predict` calls and tool calls.
- The usage of each model call from the LM's history: prompt, completion, and reasoning tokens, the
  response model, and LiteLLM's cost (0.0 on 8 of 11 calls, null on 3).
- The exception type of a failed tool call, `TimeoutError`, with its message.
- From the modules: each agent's tools, `finish` included, and each LM's temperature and output limit.
  The LM's settings also hold the API key, so the app logs only the sampling settings.

## Declared by the application

DSPy has no routing, handoff, memory, or approval construct, so the app reports these in its event log,
each with the id of the call it happened in, when there is one (the handoff and the verdicts have
none):

- the routing decision: candidates, selected agents, and the coordinator's answer as the reason;
- the handoff from the coordinator to the writer;
- the write and reads of the `notes` memory, inside the `save_note` and `read_note` tool calls, with the
  written value;
- each evaluator verdict and its reason (the evaluator loop is application code);
- the person's approval, inside the `send_brief` call before it writes the file;
- the target of the effect.

## Neither path

- The tool call id. `ReAct` does not use the provider's tool calls: the model writes the tool name and
  arguments as fields of its answer, and the record links each turn's model output to the tool call
  that follows it in the same `ReAct` call, and each tool result to the later model calls of that call.
- The retry of a failed tool call. `ReAct` puts the error into the trajectory and the model calls the
  tool again; the record links the two calls because they have the same tool and the same arguments.
- What an answer came from. An agent's answer is linked to the model output that contains it, a model
  input to earlier agent answers it contains word for word, the note to the facts agent's answer, and the
  arguments of a tool call to the latest earlier answer they contain, all matched by content (knownBy
  inferred).
- The time a memory record's content is about.
