# MCP and A2A: what each path delivers

Checked against run `pr-20261002T162420` (mcp 2.2.0, a2a-sdk 1.2.1, openai 3.23.0,
opentelemetry-instrumentation-openai-v2 2.3b0 with opentelemetry-util-genai 1.2b0, opentelemetry-sdk
1.45.0; model google/gemma-4-26b-a4b-it).

The coordinator is an A2A client. The weather, facts, translator, and writer agents are A2A servers,
each with its own Agent Card, and the weather, facts, and writer agents call one MCP server as MCP
clients. Everything runs in one process: A2A goes over in-process HTTP (an ASGI transport, no open
port), and MCP over the SDK's in-memory transport with the handshake-era protocol (version
2025-11-25), which keeps JSON-RPC framing and server-to-client requests. Each A2A request starts from
an empty trace context, as it would in another process, because A2A carries no trace context. Model
calls go to OpenRouter through the OpenAI client, traced by the official OpenTelemetry instrumentation
of that client. Version 2.4b0 of that instrumentation imports a module that util-genai 1.2b0, the
latest release, does not have, so the app uses 2.3b0 with wrapt 1.

## Spans

From the MCP SDK, which traces by default:

- One server span per request the server receives, with `mcp.method.name`, `mcp.protocol.version`,
  and `jsonrpc.request.id`. A `tools/call` span adds `gen_ai.operation.name` `execute_tool` and
  `gen_ai.tool.name`; a failed call has status ERROR and `error.type` `tool_error`. The spans carry no
  arguments, no result, and no client name.
- One client span per request sent, named `MCP send <method> <name>`. The client puts the W3C trace
  context in the request's `_meta`, so the server span of every request is a child of the client span
  that sent it. The `notifications/initialized` notifications have no client span.
  A request the server sends to the client, such as `elicitation/create`, gets its own client span
  under the server span of the tool call that sent it.

From the A2A SDK: 342 spans, one per call of a traced SDK method (request handler, JSON-RPC
dispatcher, event queue, client transport), all with no attributes. Six of them have status ERROR:
the `QueueShutDown` that ends each task's event queue. No span names a task, an agent, or a message.

From the OpenAI instrumentation: one `chat` span per model call, with the request and response
model, temperature, max tokens, response id, finish reasons, and token counts. `gen_ai.system` is
`openai`, the client library; nothing names OpenRouter. The spans carry no message content.

## Only the instrumentation's log events

This version of the OpenAI instrumentation writes the content as OpenTelemetry log events, each with
the span id of its model call: the system, user, assistant, and tool messages of the input, and the
choice. The choice holds the id of every tool call the model requested, and a tool message holds the
id of the call whose result it carries.

## Only the protocol messages

What the MCP server received, logged by a middleware inside the SDK's server span:

- the arguments and the result of each tool call, with `isError`;
- the hints each tool declares in `tools/list`: `get_weather` and `search_facts` are read-only,
  `save_note` is neither read-only nor destructive, and `send_brief` is destructive;
- the client name each agent gives at `initialize`, which ties every later request to its agent, and the
  server's name and version in the answer, which the record gives to each of its tools;
- the resource link that `save_note` returns (`notes://Lisbon`) and the contents a `resources/read`
  returns;
- the elicitation of `send_brief`, with its message and schema, and the person's answer, which the
  client callback logs; its trace context names the elicitation's client span, under the server
  span that sent it.

What the A2A client sent and received, logged from the HTTP messages:

- each Agent Card, with the agent's name, description, and skills;
- each `SendMessage` request, with the message id, the context id, and the text;
- each response, with the task id, its final state, the status timestamp, the artifacts, and the
  history.

## Only the application's own objects

- The cost of each model call, from the usage block of OpenRouter's response, with the response id
  that names the model call's span.
- The task each agent turn served and the trace it ran in, logged when the agent's executor starts.
  This joins an agent's model calls and MCP calls to its task.

## Declared by the application

- The routing decision: the candidates among the resolved Agent Cards and the selected agents. Its router
  is the A2A client, which the A2A messages do not name. The application keeps only the weather and
  facts agents from the coordinator's choice, so the selection is the application's.
- Each evaluator verdict and its reason (the evaluator loop is application code).
- The model's tool call id, which the application puts in the `_meta` of each MCP `tools/call`
  request, so the server's record of the call names the model output that requested it.

## Neither path

- The original error of a failed tool. The server turns the `TimeoutError` into the text "Error
  executing tool get_weather" and the span into `error.type` `tool_error`.
- The retry of a failed tool call. The record links the next call of the same tool by the same
  client to the failed one.
- A message that says which resource a tool changed. The SDK's server answers `resources/subscribe`
  with "Method not found" (the 2026-07-28 revision removed subscriptions), so the converter reads a
  resource link returned by a tool that is not read-only as a write of that resource.
- That a tool's result changed something outside the run. The converter treats the result of a tool
  that declares itself destructive as an effect, with the result text as its target.
- The service that answered the model calls, OpenRouter.
- What an artifact, a message, or a memory record came from. The record links an artifact to the
  agent's last answer with the same text, and a message or a model input to the earlier answers and
  artifacts it holds word for word (knownBy inferred).
