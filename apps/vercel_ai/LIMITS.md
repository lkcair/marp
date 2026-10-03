# Vercel AI SDK: what each path delivers

Checked against run `va-20261002T155631` (ai 7.0.127, @ai-sdk/otel 1.0.127, @openrouter/ai-sdk-provider
3.1.0, @opentelemetry/sdk-trace-base 2.11.0, @opentelemetry/context-async-hooks 2.11.0, zod 4.6.5, Node
22.22; model google/gemma-4-26b-a4b-it through OpenRouter).

The app is written with the SDK's `ToolLoopAgent`. A coordinator agent has the weather, facts, and
translator agents as tools, each a tool whose `execute` runs that agent, and the SDK runs the tool calls of
one model step concurrently, so the weather and facts agents ran in parallel. The writer, the evaluator,
and the sender are agents the app calls in turn; the evaluator loop is application code. The sender's
`send_brief` tool has `toolApproval: { send_brief: "user-approval" }`, so the first call returns a
tool approval request and a second call with the person's response runs the tool.

In version 7 the SDK emits telemetry only through a registered integration. The app registers
`@ai-sdk/otel` with a tracer of its own whose spans go to a local file, and an async context manager so
that a subagent's spans nest under the tool call that runs it. No span left the machine.

## Spans (@ai-sdk/otel, OpenTelemetry GenAI conventions)

- One `invoke_agent` span per agent call, with `gen_ai.agent.name` taken from `telemetry.functionId`. A
  subagent's `invoke_agent` span is a child of the `execute_tool` span of the call that ran it. Each
  top-level agent call is its own trace: the run has six traces and nothing joins them.
- One `agent_step` span per step of an agent's loop, named `step N`.
- One `chat` span per model call: request and response model, provider `openrouter`, temperature, output
  token limit, response id, finish reasons, input, output, cache read, and cache creation tokens, the
  system instructions, the tool definitions, and the input and output messages. The messages use the
  GenAI parts format and carry the id of every tool call the model requested and of every tool result it
  read, so model calls and tool calls link by id.
- One `execute_tool` span per executed tool call: tool name, call id, tool type, arguments, result, and
  duration. The failed call has status ERROR, the error message, and an exception event of type
  `TimeoutError`.
- No conversation id and no cost.

## Only the SDK's callbacks, results, and objects

- Each agent call's `callId`, which no span carries; the record joins agent calls to their spans by
  agent name and order.
- The response id of each model call (`onLanguageModelCallEnd`), equal to the chat span's response id, and
  the tool call id of each tool execution (`onToolExecutionStart` and `onToolExecutionEnd`), equal to the
  tool span's call id, so the two paths join exactly.
- The cost of each model call and the provider behind OpenRouter (NextBit for 13 calls, SiliconFlow for
  one), from the provider metadata that OpenRouter returns with `usage: { include: true }`.
- The tool approval request, with its approval id and the held tool call. A held call is not executed and
  has no span; the approved call runs on the second call, before that call's first step, so its
  `execute_tool` span sits directly under the agent span.
- From the agent objects, the tools each agent can call.

## Declared by the application

- Which of the coordinator's tools run agents. The SDK sees them as ordinary tools; the app names them
  from its own list, so that the translator, a candidate that did not run, is an agent in the record.
- The handoff from the coordinator to the writer: the SDK has no handoff, so the app passes the
  coordinator's answer to the writer.
- The write and reads of the `notes` memory, with the written value.
- Each evaluator verdict and its reason.
- The person's approval decision, which the app gives as the approval response.
- The target of the effect.

## Neither path

- A session: no span or callback names a conversation.
- The retry of the failed tool call. The SDK returns the error to the model, which calls the tool again;
  the record links the two calls because they have the same tool and the same arguments.
- What a tool result or a model input came from. The result of a subagent's tool call is linked to the
  subagent's final answer because their contents are equal, and a model input that holds an earlier
  answer word for word, at least 16 characters long, is linked to it (knownBy inferred).
- The time a memory record's content is about.
