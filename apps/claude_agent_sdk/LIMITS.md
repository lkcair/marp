# Claude Agent SDK: what each path delivers

Checked against run `cc-20261002T072558` (claude-agent-sdk 0.2.163 with its bundled Claude Code CLI
2.1.286; model nvidia/nemotron-3.5-lightning:free through OpenRouter's Anthropic-compatible endpoint).

The SDK drives the Claude Code CLI as a subprocess. The app points the CLI at OpenRouter with
`ANTHROPIC_BASE_URL`, `ANTHROPIC_AUTH_TOKEN`, and the model id, and gives it a fresh configuration
directory so no user setting, memory, or hook is loaded. The CLI exports OpenTelemetry traces when
`CLAUDE_CODE_ENABLE_TELEMETRY=1`, `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1`, and `OTEL_TRACES_EXPORTER=otlp`
are set, and log events with `OTEL_LOGS_EXPORTER=otlp`. Both go to an OTLP/HTTP receiver inside the
app, so nothing left the machine. Subagents run in the foreground only with
`CLAUDE_CODE_DISABLE_BACKGROUND_TASKS=1`; without it this CLI version launched them in the background
and the coordinator ended before they answered.

## Spans (Claude Code enhanced telemetry, beta)

The spans use the CLI's own names and do not set `gen_ai.operation.name`, so a reader of the OTel
GenAI conventions sees them as plain steps.

- One `claude_code.interaction` span per prompt, with the session id, the sequence of the prompt in its
  session, and the prompt text. Each interaction is its own trace; the session id joins them.
- One `claude_code.llm_request` span per model call: request model, `gen_ai.system` (`anthropic`, the
  protocol, although OpenRouter answered), response id, finish reasons, input, output, and cache tokens,
  and the attempt number. No message content.
- One `claude_code.tool` span per tool call, with the tool name and the call id
  (`gen_ai.tool.call.id`). A call of the `Agent` tool names the subagent it starts (`subagent_type`).
  Under it, `claude_code.tool.execution` has success or failure and the error message, and
  `claude_code.tool.blocked_on_user` has the permission decision when it is a rejection. No arguments;
  the result appears only as a `tool.output` span event, on 9 of the 18 calls.
- The model and tool calls of a subagent nest under the execution span of the `Agent` call that started
  it, and carry the subagent's id (`agent_id`), so the spans show two subagents running at the same
  time.

## OpenTelemetry log events

- `claude_code.api_request` for each model call, with the request id of the span, tokens, and
  `cost_usd`. The CLI computes this cost itself from its price table; OpenRouter charges nothing for a
  free model, so the figure is the CLI's estimate.
- `claude_code.assistant_response` with the text of each answer, including the final answer of a
  subagent (with `OTEL_LOG_ASSISTANT_RESPONSES=1`).
- `claude_code.tool_result` with the tool input and parameters (with `OTEL_LOG_TOOL_DETAILS=1`), and
  `claude_code.tool_decision` with each permission decision and its source: `config` for a tool the
  options allow, `user_temporary` for a call the permission callback allowed, and `user_reject` for one
  it denied.
- `claude_code.user_prompt` and `claude_code.subagent_completed`.

## Only the native API

- The message stream: each model answer with its message id (equal to the span's response id), its tool
  calls with their ids and inputs, the parent tool call of a subagent's message, and the result of each
  session. The final text of a subagent reaches the stream only when `forward_subagent_text` is set; the
  app reads it from the log above.
- The agents each session can start, listed in the session's first message. They are the candidates of
  the coordinator's routing: the three agents of the task and the CLI's built-in agents.
- The arguments and result of every tool call, and the error of a failed one, from the `PreToolUse`,
  `PostToolUse`, and `PostToolUseFailure` hooks, keyed by the call id.
- The start and end of each subagent, with its id and type, from the `SubagentStart` and
  `SubagentStop` hooks. The CLI starts a subagent afresh for each `Agent` call, so the record says which call
  created it.
- The permission callback (`can_use_tool`) with the call id, which the app answers for the person who
  approves `send_brief`. The same callback lets only subagents call the worker tools; the coordinator's
  own attempts to call them were denied and are recorded as blocked.

## Declared by the application through its event log

- the handoff from the coordinator to the writer: the SDK has no handoff, so the app starts the writer's
  session with the coordinator's answer;
- reads and writes of the `notes` memory, written by a hook from the facts agent's answer;
- each evaluator verdict and its reason (the evaluator loop is application code: the application sets the
  first verdict, and each later verdict comes from its own evaluator session);
- the approval decision of the person;
- the target of the effect.

## Neither path

- The model input. Neither the spans nor the stream carry the prompt the CLI sends; the record links
  each model call to the previous answer of the same session or subagent, since the CLI sends the whole
  transcript every time.
- The exception the tool raised. The `TimeoutError` reaches the record as `McpToolCallError` with its
  message.
- The retry of a failed tool call. The CLI returns the error to the model, which calls the tool again;
  the record links the two calls because they have the same tool and the same arguments.
- What a tool result came from. The result of an `Agent` call is linked to the subagent's final answer
  because their contents are equal (knownBy inferred).
- The sampling settings of a model call. The SDK has no temperature option and the CLI reports none,
  so a replay of this run cannot recover them.
- The service that answered the model calls and what it charged.
- The time a memory record's content is about.
