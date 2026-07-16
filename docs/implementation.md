# Worldbuilder 0.1 (MVP) Implementation Design

**Status:** implementation plan for version 0.1, which is identical to the MVP defined in [mvp.md](mvp.md)  
**Last researched:** 2026-07-16  
**Language/runtime:** Python `>=3.14`  
**Direct runtime dependencies:** `openai>=2.45,<2.46`, `textual>=8.2.8,<9` (initial reviewed ranges; `uv.lock` pins exact resolved versions)

## 1. Implementation decisions

Version 0.1 (the MVP) will implement its own bounded harness loop over the OpenAI Responses API. It will not use the OpenAI Agents SDK or a general agent framework. This gives Worldbuilder direct ownership of workspace safety, chat/session persistence, approval suspension/resumption, event persistence, model compatibility, and the distinction between worldbuilding facts and proposals.

The two presentation surfaces are adapters:

- standard-library `argparse` CLI for scripting and simple terminal interaction;
- Textual TUI for durable named chats, interactive runs, event display, diffs, and approval.

Both call the same chat and run application services. Neither surface calls OpenAI, reads/writes world files, executes tools, or decides policy directly.

Version 0.1 has no Git adapter and runs no Git command. Internal `ChangeSet` records are application-owned write proposals stored in SQLite; they are not Git changes, commits, staging entries, or branches.

Version 0.1 has no web-search, browser, scraper, or generic HTTP tool. Its only required network boundary is the OpenAI API adapter. A user request for current online research must receive a clear capability limitation rather than causing an invented or untracked network action.

OpenAI integration uses:

- `client.responses.create(...)` as the model endpoint;
- custom strict function tools for local reads/writes;
- `reasoning={"effort": ...}` for effort selection; and
- the response output item sequence as the provider-neutral input to the harness state mapper.

## 2. Target package structure

```text
src/worldbuilder/
  __init__.py
  bootstrap.py
  cli/
    app.py
    parsing.py
    presenter.py
    confirmation.py
  tui/
    app.py
    controller.py
    command_parser.py
    command_handlers.py
    command_palette.py
    messages.py
    widgets/
      transcript.py
      text_input.py
      choice_list.py
    worldbuilder.tcss
  domain/
    identifiers.py
    chats.py
    runs.py
    tools.py
    changes.py
    policies.py
    errors.py
  application/
    facade.py
    commands.py
    results.py
    runtime/
      runner.py
      state.py
      budgets.py
      events.py
      continuation.py
      context_builder.py
    services/
      approvals.py
      change_sets.py
      workspaces.py
      chats.py
      workspace_lease.py
  ports/
    model_gateway.py
    tool_executor.py
    run_repository.py
    chat_repository.py
    workspace_repository.py
    approval_gateway.py
    event_sink.py
    clock.py
  infrastructure/
    openai/
      client.py
      gateway.py
      mapper.py
      model_profiles.py
      errors.py
    filesystem/
      paths.py
      reader.py
      writer.py
      atomic.py
      locking.py
    sqlite/
      connection.py
      migrations.py
      run_repository.py
      chat_repository.py
      change_repository.py
    config.py
  tools/
    registry.py
    read_text_file.py
    write_markdown_file.py
  prompts/
    system.md
    renderer.py
tests/
  unit/
  integration/
  acceptance/
  fixtures/
```

The exact module split may be reduced while code is small. The dependency direction is mandatory: `domain` and `application` do not import `openai`, `textual`, CLI, SQLite, or filesystem implementations.

## 3. Core types

Use frozen standard-library dataclasses, enums, and protocols for application contracts. External/provider data is parsed explicitly before it enters the core.

### 3.1 Run request

```python
@dataclass(frozen=True, slots=True)
class AgentRunRequest:
    run_id: RunId
    chat_id: ChatId
    user_item_sequence: int
    workspace: WorkspaceRef
    model_id: str
    reasoning_effort: str
    approval_mode: ApprovalMode
    capabilities: CapabilityPolicy
    budgets: RunBudgets
    retention: RetentionPolicy
```

`StartRun(user_input, ...)` validates and persists the message/run first; `AgentRunRequest` deliberately contains only its sequence reference, not a second text copy. Validation occurs in that factory/application command before this object is created:

- input is not blank;
- input is strict UTF-8 without NUL/isolated surrogates and at most 256 KiB encoded;
- workspace exists and resolves safely;
- model/effort combination exists in `ModelProfileRegistry`;
- required tool capabilities are supported;
- budgets are positive and below configured hard maxima, with `max_local_tool_calls < max_model_turns` because 0.1 permits one call per turn and must reserve a possible final-answer turn; and
- chat, write-approval mode, and all effective settings are explicit.

The request contains an immutable snapshot. Changing a chat default while a run is active affects only the next run.

### 3.2 Chat state

```python
@dataclass(frozen=True, slots=True)
class Chat:
    chat_id: ChatId
    workspace_id: WorkspaceId
    name: str
    model_id: str
    reasoning_effort: str
    approval_mode: ApprovalMode
    created_at: datetime
    updated_at: datetime
    last_opened_at: datetime

class ApprovalMode(StrEnum):
    ASK = "ask"
    DONT_ASK = "dont-ask"
```

Chat IDs, not names, are foreign keys. Preserve the trimmed display text, validate 1–120 Unicode code points, and reject control and bidirectional-formatting characters. Derive uniqueness keys with `unicodedata.normalize("NFKC", name).casefold()` and enforce them per workspace. Renaming changes no messages, runs, or continuation state.

### 3.3 Run state

```python
class RunStatus(StrEnum):
    CREATED = "created"
    RUNNING = "running"
    AWAITING_APPROVAL = "awaiting_approval"
    COMPLETED = "completed"
    CANCELLED = "cancelled"
    BUDGET_EXCEEDED = "budget_exceeded"
    CONTEXT_LIMIT_REACHED = "context_limit_reached"
    POLICY_BLOCKED = "policy_blocked"
    REFUSED = "refused"
    PROVIDER_FAILED = "provider_failed"
    TOOL_FAILED = "tool_failed"
    FAILED = "failed"
    INTERRUPTED = "interrupted"

@dataclass(frozen=True, slots=True)
class RunState:
    run_id: RunId
    chat_id: ChatId
    status: RunStatus
    model_turns: int
    local_tool_calls: int
    write_proposals: int
    started_at: datetime
    active_response_id: str | None
    continuation_items: tuple[ProviderContinuationItem, ...]
    pending_call: PendingToolCall | None
    pending_approval: ApprovalRequest | None
    final_text: str | None
```

State changes are created by transition functions; each database state mutation and its corresponding event commit in the same SQLite transaction. Filesystem/provider boundaries use explicit before/after journal records because they cannot share that database transaction.

### 3.4 Mapped output and lossless continuation items

The OpenAI adapter maps user-visible/actionable SDK output to a closed local union:

```text
ModelItem =
  AssistantText(text)
  FunctionCall(call_id, name, arguments_json)
  FunctionResult(call_id, name, result_json)
  ReasoningSummary(summary_text)        # only public API-provided summary
  Refusal(reason)
  UnknownProviderItem(provider_type, safe_metadata)
```

Never pass OpenAI SDK objects into domain/application code. Separately, the adapter serialises every API output item needed for stateless continuation into a tagged `ProviderContinuationItem(provider="openai", api_schema_version, payload_json)` without lossy remapping. This includes encrypted reasoning content and the complete compacted window returned with `store=false`. These opaque items are encrypted/serialised continuation state, not private chain-of-thought to display or interpret; they are stored in protected `context_json` only. Unknown future output items are never silently dropped from a continuation: the adapter either proves they are presentation-only or fails the turn with `PROVIDER_OUTPUT_UNSUPPORTED` before any tool executes.

### 3.5 Events

Events are public application evidence, not model chain-of-thought:

```text
RunCreated
RunStarted
ChatCreated
ChatRenamed
ChatOpened
ChatItemAppended
ContextPrepared
ModelRequestStarted
ModelResponseReceived
AssistantTextDelta
AssistantTextCompleted
FunctionCallRequested
FunctionCallValidated
PolicyDecisionMade
ApprovalRequested
ApprovalDecided
ToolExecutionStarted
ToolExecutionCompleted
ToolExecutionFailed
WriteAutoApplyStarted
WriteApplied
WriteFailed
RunRefused
RunCancelled
BudgetExceeded
ContextLimitReached
RunProviderFailed
RunToolFailed
RunCompleted
RunFailed
```

Each durable event has a chat ID where applicable, optional run ID, monotonic per-stream sequence, UTC timestamp, schema version, payload, and redaction level. `AssistantTextDelta` is explicitly ephemeral (`durable=false`) and uses a separate in-memory stream sequence; it is never replayed as if committed. UI rendering consumes these event types rather than provider stream classes.

## 4. Configuration and model profiles

### 4.1 Configuration sources

Implement standard-library TOML reading for application defaults and optional workspace settings. Version 0.1 does not rewrite or generate `worldbuilder.toml`; `init` creates only reserved `.worldbuilder/` operational state. A user who wants workspace overrides creates the documented TOML file explicitly, avoiding an implicit write to user configuration and the need for a TOML writer.

Model, effort, and approval-mode precedence:

```text
explicit CLI option
  > persisted current-chat slash-command/default value
  > WORLDBUILDER_MODEL / WORLDBUILDER_EFFORT / WORLDBUILDER_APPROVAL
  > workspace configuration
  > packaged internal defaults
```

All three environment variables are optional. The CLI options `--model`, `--effort`, and `--approval` are optional. Explicit options override only the current run unless the user changes a chat default through the TUI command. A new chat resolves and persists each default together with its original source; the persisted value then outranks later environment/workspace changes until explicitly changed. Invalid values in any configured source are errors even when shadowed by a higher-precedence value, so an unsafe latent value cannot become active unexpectedly.

Resolve the model before effort because effort validity depends on the selected model:

```python
model = first_not_none(
    run_options.model,
    current_chat.model_id if current_chat else None,
    environ.get("WORLDBUILDER_MODEL"),
    workspace_config.model,
    INTERNAL_DEFAULT_MODEL,
)
profile = model_profiles.require(model)

effort = first_not_none(
    run_options.effort,
    current_chat.reasoning_effort if current_chat else None,
    environ.get("WORLDBUILDER_EFFORT"),
    workspace_config.effort,
    profile.default_effort,
)
profile.require_effort(effort)

approval = first_not_none(
    run_options.approval,
    current_chat.approval_mode if current_chat else None,
    environ.get("WORLDBUILDER_APPROVAL"),
    workspace_config.approval,
    ApprovalMode.ASK,
)
approval = ApprovalMode(approval)
```

Return `EffectiveSetting(value, source)` rather than only a string so CLI output, `/status`, and run persistence can explain where the effective value came from. Use `os.environ` directly; do not add a dotenv/settings dependency to version 0.1.

Suggested non-secret workspace file:

```toml
[model]
id = "gpt-5.6"
effort = "medium"

[writes]
approval = "ask"

[budgets]
max_model_turns = 12
max_local_tool_calls = 10
max_output_tokens = 16384
timeout_seconds = 300

[provider.openai]
request_timeout_seconds = 60

[storage]
max_backup_bytes = 1073741824  # 1 GiB; no automatic purge
```

Reject unknown keys; warnings are insufficient for misspelled security, budget, or provider settings. `timeout_seconds` is accumulated active run time, while `request_timeout_seconds` caps one SDK attempt and is always further reduced to remaining run time. `OPENAI_API_KEY` is read separately and never included in the merged printable configuration.

### 4.2 Model profile registry

Capabilities cannot be inferred reliably from a model ID string. Maintain a small reviewed registry:

```python
@dataclass(frozen=True, slots=True)
class ModelProfile:
    id: str
    aliases: tuple[str, ...]
    reasoning_efforts: frozenset[str]
    default_effort: str
    responses: bool
    function_calling: bool
    context_window_tokens: int
    compact_threshold_tokens: int

PROFILES = {
    "gpt-5.6": ModelProfile(
        id="gpt-5.6",
        aliases=(),
        reasoning_efforts=frozenset(
            {"none", "low", "medium", "high", "xhigh", "max"}
        ),
        default_effort="medium",
        responses=True,
        function_calling=True,
        context_window_tokens=VERIFIED_GPT_5_6_CONTEXT_WINDOW,
        compact_threshold_tokens=REVIEWED_GPT_5_6_COMPACT_THRESHOLD,
    ),
    # Add lower-cost profiles only after verifying their current model pages.
}
```

The documented effort set was checked against the current GPT-5.6 guide on 2026-07-16, but must be reverified when the lock/model profile changes. The profile registry is versioned and covered by a test that asserts the default model supports every required 0.1 capability. Do not query `/models` on every run: model-list presence alone is not sufficient proof of function-calling and effort compatibility.

### 4.3 CLI and TUI selection

CLI parsing accepts optional strings but application validation returns structured errors. When no flags are supplied, the resolver uses persisted current-chat values, supported environment values, workspace values, then internal defaults. TUI model/effort options come from the registry. `/model`, `/effort`, and `/approval` change the current chat defaults in SQLite; changing the model immediately refreshes valid effort choices and resets an invalid prior effort to that model's default with a visible transcript notice.

The effective model, effort, and approval mode are frozen into `AgentRunRequest`; settings changes during an active run apply only to the next run. `Ask for Approval` maps to `ask`; `Dont ask for Approval` maps to `dont-ask`. Dont-ask is a standing per-chat permission. Emit a prominent warning before every run whose effective snapshot is Dont-ask, irrespective of source, stating that prompt-injected workspace text could influence the one automatically applied valid write; keep it visible in `/status` and recommend Ask for untrusted corpora. The command handler also warns when persisting it. The model has no tool or command that can change this field.

## 5. OpenAI Responses adapter

### 5.1 Client lifecycle

Construct one OpenAI client in the composition root and inject it into `OpenAIResponsesGateway`. Do not instantiate a client inside each tool or UI handler. Set SDK automatic retries to zero; the harness owns every retry so attempts, backoff, cancellation, request IDs, and total deadlines are observable and budgeted exactly once. Configure the per-attempt transport timeout centrally, always capped by remaining run time.

The adapter is the only code allowed to import `openai`.

```python
class OpenAIResponsesGateway(ModelGateway):
    def __init__(self, client: AsyncOpenAI, mapper: OpenAIResponseMapper): ...

    async def create_turn(self, request: ModelTurnRequest) -> ModelTurn: ...
```

Prefer `AsyncOpenAI` because Textual and the harness use `asyncio`. The CLI calls the async application entry point via `asyncio.run()`.

### 5.2 First request

Conceptual request shape:

```python
response = await client.responses.create(
    model=request.model_id,
    reasoning={"effort": request.reasoning_effort},
    instructions=prompt_renderer.system_instructions(),
    tools=[
        *tool_registry.openai_function_definitions(policy),
    ],
    tool_choice=request.tool_choice,  # "auto" normally; "none" on reserved final turn
    parallel_tool_calls=False,
    truncation="disabled",
    max_output_tokens=request.budgets.max_output_tokens,
    input=context_builder.for_new_run(
        request.chat_id, through_sequence=request.user_item_sequence
    ),
    store=False,
)
```

Verify the exact SDK field signatures against the pinned SDK version during implementation. The implementation deliberately uses `store=False` and manages continuation locally for privacy and durable local ownership. Current OpenAI documentation states that stateless reasoning output contains `encrypted_content` by default; the adapter must retain that field losslessly and replay all output items required by the API. If the pinned SDK/API cannot round-trip the required encrypted reasoning/compaction fields, version 0.1 fails its live contract test and must not silently switch to provider-side storage.

### 5.3 Tool definition mapping

Custom functions use strict JSON Schema. OpenAI recommends strict mode and requires all properties to be required and `additionalProperties: false`; nullable types represent optional fields. [OpenAI: Function calling — strict mode](https://developers.openai.com/api/docs/guides/function-calling)

Tool registry output example:

```json
{
  "type": "function",
  "name": "read_text_file",
  "description": "Read a bounded line range from a UTF-8 text file inside the active workspace.",
  "strict": true,
  "parameters": {
    "type": "object",
    "properties": {
      "path": {"type": "string"},
      "start_line": {"type": ["integer", "null"]},
      "end_line": {"type": ["integer", "null"]}
    },
    "required": ["path", "start_line", "end_line"],
    "additionalProperties": false
  }
}
```

Schema compliance is not authorization. The local parser validates again, and the policy service checks the resulting typed request.

### 5.4 Mapping output

For each complete response:

1. Record provider response ID, requested model alias, provider-returned concrete model identifier when supplied, status, usage, and safe error metadata.
2. Iterate `response.output` in order.
3. Preserve every continuation-relevant raw output item in a lossless, locally serialisable provider envelope before executing any function call; redacted event metadata is a separate representation and must not replace it.
4. Map messages/output text and public annotations.
5. Map every `function_call` to a typed `FunctionCall`; JSON-decode arguments only after recording the original safe payload/digest.
6. Preserve the complete compacted output window, including encrypted compaction/reasoning fields and retained items, without displaying or interpreting opaque fields.
7. Map refusals/incomplete states explicitly.
8. Determine whether the turn contains local calls or a final assistant result.

Do not rely only on `response.output_text`, because it omits tool-call structure needed by the harness.

### 5.5 Continuation strategy

The function-call guide defines the loop as returning `function_call_output` with the matching `call_id`, and notes that reasoning items returned alongside tool calls must be sent back with tool results. [OpenAI: Function calling](https://developers.openai.com/api/docs/guides/function-calling)

Therefore the continuation builder shall:

```python
continuation.extend(mapped_provider_output_as_input_items(response.output))
continuation.append({
    "type": "function_call_output",
    "call_id": call.call_id,
    "output": json.dumps(tool_result.to_wire(), ensure_ascii=False),
})
```

Then create the next response with the same model, effort, instructions, and tool definitions. Passing the entire required response output is essential; passing only assistant text or only the function call can break reasoning-model continuation.

Provider-side `previous_response_id` is a possible later optimisation. OpenAI documents it for linked turns, but response objects are stored for 30 days by default and previous input tokens remain billed. Conversations API items persist without the response object's 30-day TTL. [OpenAI: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state) Local continuation with `store=False` is the 0.1 design because named chats, retention control, switching, and resumable approval must remain locally authoritative. Between runs, the context builder creates one projection ending at the already persisted `user_item_sequence`; `AgentRunRequest` has no duplicate message-text field to append. Within a run, the persisted continuation ledger is the sole source for the next provider turn. These rules prevent duplicate user messages, tool calls, or outputs.

For a long chat, `ContextBuilder` composes a bounded request from the most recent committed chat items while preserving every unresolved function-call/output pair. It uses the model profile's reviewed context limit and a conservative UTF-8-byte upper bound plus an output/tool/instruction safety reserve; it never asks the API to auto-truncate. A reviewed profile value sets `context_management=[{"type": "compaction", "compact_threshold": N}]`, with `N` below the context limit by at least the full reserved output/instruction/tool margin. When compaction triggers under `store=False`, persist the complete returned compacted window as provider continuation chat items. Future requests pass that window back unchanged and drop only items that precede its latest compaction boundary as the official guide permits. The full user-visible transcript remains local and is never replaced by opaque continuation state. [OpenAI: Compaction](https://developers.openai.com/api/docs/guides/compaction)

### 5.6 Streaming

Implement correctness with non-streaming complete responses first during development, then add and acceptance-test streaming before version 0.1 release. Interactive CLI/TUI runs stream by default; CLI `--no-stream` selects complete-response mode. Both modes produce the same durable complete items, tool effects, budgets, and terminal result:

- provider deltas become temporary public progress/assistant-delta events;
- only the completed response object is parsed for tool execution and durable continuation;
- a partial streamed function argument is never executed;
- the UI may show “preparing tool call” but waits for a complete validated call;
- stream interruption produces `PROVIDER_FAILED`/`INTERRUPTED`, not a guessed final output.

The UI labels deltas as provisional. Only `AssistantTextCompleted` from the validated complete response is appended to durable chat history. After a crash, transcript reconstruction omits provisional deltas and shows the persisted interrupted/failure state, so vanished partial text cannot be mistaken for a committed assistant message.

This delivery sequence reduces implementation risk; non-streaming alone is not completion while the normative `--no-stream` option and default-stream UX remain in scope.

## 6. Harness loop

### 6.1 Runtime algorithm

```python
async def run(request: AgentRunRequest) -> RunResult:
    state = repository.start_created_run_and_event(request.run_id)
    continuation = context_builder.for_new_run(
        request.chat_id, through_sequence=request.user_item_sequence
    )

    while True:
        cancellation.raise_if_requested()
        state = repository.reserve_model_attempt(state, budgets, clock)

        turn = await model.create_turn(
            build_turn_request(
                request,
                continuation,
                tool_choice=budgets.tool_choice_for_remaining(state),
            ),
            timeout=budgets.remaining_deadline(state),
        )
        repository.record_model_turn(state.run_id, turn)

        emit_public_provider_events(turn)

        if turn.refusal is not None:
            return terminate_refused(turn)

        calls = turn.function_calls
        if len(calls) > 1:
            return terminate_provider_failed("OPENAI_OUTPUT_INVALID")
        if not calls:
            return complete_with_validated_final_text(turn)

        continuation.extend(turn.continuation_items)

        for call in calls:
            state = repository.reserve_tool_attempt(state, budgets, clock)
            validation = registry.parse(call)
            if not validation.ok:
                result = validation.to_tool_error()
                repository.record_tool_result(call, result)
                continuation.append(function_output(call.call_id, result))
                continue
            parsed = validation.value
            decision = policy.decide(request.capabilities, parsed)
            repository.record_policy_decision(decision)

            if decision.denied:
                return terminate_policy_blocked(decision)

            if parsed.effect is ToolEffect.WRITE:
                prepared = await change_sets.prepare(parsed, decision.prepare_grant)
                if not prepared.ok:
                    result = prepared.to_tool_error()
                    repository.record_tool_result(call, result)
                    continuation.append(function_output(call.call_id, result))
                    continue
                if decision.requires_approval:
                    pending = approval_service.create_exact_request(
                        prepared.change_set_id, prepared.digest
                    )
                    repository.suspend_for_approval(
                        state, pending, continuation, call
                    )
                    return RunResult.awaiting_approval(pending)
                result = await change_sets.apply_auto(prepared.change_set_id)
            else:
                result = await executor.execute(parsed, decision.execution_grant)
            repository.record_tool_result(call, result)
            continuation.append(function_output(call.call_id, result))
```

The `StartRun` application command first validates input/settings, then appends the one user chat item and inserts the `CREATED` run referencing its sequence plus `RunCreated` in one transaction. Only then does it construct `AgentRunRequest` and call the runner, which compare-and-sets that row to `RUNNING` with `RunStarted`; the runner never inserts user text or a second run. Reservation and its event commit before each external attempt, so a crash or failure still consumes the budget and the maximum cannot be exceeded by one. Schema/argument parsing returns a typed validation result rather than throwing past the loop: a safely recoverable `TOOL_ARGUMENT_INVALID` is persisted and appended as that call's `function_call_output`, while an internal parser invariant terminates `FAILED`. An explicit policy denial terminates `POLICY_BLOCKED` and never reaches the executor. Write preparation performs all validation, stale read, exact-byte encoding, diff/digest creation, and `PREPARED` persistence before the approval branch; it does not mutate the target. Thus `create_exact_request` is always bound to an actual persisted diff, not merely model arguments.

Run state tracks unresolved failed observations by deterministic `failure_scope_key`: `(tool_name, canonical_target)` when a safe target was parsed, otherwise `(tool_name, null)`. A later confirmed success clears the matching target key and that tool's targetless validation key; approval rejection is a user decision and never sets either. On final assistant text, `complete_with_validated_final_text` requires provider status `completed`, no tool call/refusal, and non-empty public text, then returns `TOOL_FAILED` if any unresolved key remains or `COMPLETED` otherwise. Final prose may explain failure but cannot turn it into success. Missing/invalid final structure terminates `PROVIDER_FAILED/OPENAI_OUTPUT_INVALID`. Text returned alongside a tool call is progress, not a final answer. The production loop wraps every provider/tool boundary in typed error mapping and an event; it must not catch `Exception` and turn it into model-visible success.

### 6.2 Multiple local calls

Every request sets `parallel_tool_calls=False`, which OpenAI documents as constraining custom calls to zero or one. Version 0.1 therefore defines multiple calls in one response as a provider-contract violation: persist safe diagnostics, execute none, and terminate `PROVIDER_FAILED/OPENAI_OUTPUT_INVALID`. Do not partially execute the first call or attempt to invent outputs for the rest. This fail-closed rule removes ambiguous write ordering and makes approval suspension/resumption exact. [OpenAI: Function calling](https://developers.openai.com/api/docs/guides/function-calling)

### 6.3 Tool-result envelope

Return JSON strings to the model using one stable envelope:

```json
{
  "ok": true,
  "code": "FILE_READ",
  "message": "Read 120 lines from lore/history.md.",
  "data": {},
  "retryable": false
}
```

Errors use `ok: false` and stable codes such as:

```text
PATH_OUTSIDE_WORKSPACE
PATH_INTERNAL
FILE_NOT_FOUND
FILE_TOO_LARGE
FILE_ENCODING_UNSUPPORTED
WRITE_SCOPE_DENIED
APPROVAL_REJECTED
CHANGESET_STALE
TOOL_ARGUMENT_INVALID
TOOL_BUDGET_EXCEEDED
```

Do not return Python tracebacks, absolute internal paths, secrets, or raw exception representations to the model.

### 6.4 Retry and loop detection

- Retry transient provider errors with bounded exponential backoff inside the total deadline.
- Do not automatically retry uncertain writes.
- Read retries are allowed only when no side effect occurred.
- Hash `(tool_name, canonical_arguments, error_code)` for failed calls. The first safely recoverable failure is returned to the model; a second consecutive identical failure terminates `TOOL_FAILED` immediately and is not sent into another retry turn.
- Count every attempted provider turn and local call, including failed attempts as defined by the budget policy.
- Provider-internal reasoning is counted through model turns, usage, elapsed time, and configured cost controls; it is not a local tool call.

Reserve attempts durably before the call. Accumulate active elapsed time with a monotonic clock and persist it at each boundary; time in `AWAITING_APPROVAL` is excluded. Cap every SDK attempt by `min(configured_transport_timeout, remaining_run_time)`. With one model turn left after an observation, `tool_choice_for_remaining` returns `"none"`; a custom call in that response is invalid and none executes. This reserves a truthful final-report opportunity rather than allowing a last-turn side effect. A cancellation request may cancel an outstanding model/read operation, but once a write reaches `APPLYING` it becomes `cancellation_pending`: complete or recover that filesystem journal first, then report `CANCELLED` only with the confirmed write outcome. Never abandon or label an in-flight write cancelled while its effect is unknown.

## 7. Read tool implementation

### 7.1 Typed input

```python
@dataclass(frozen=True, slots=True)
class ReadTextFileArgs:
    path: str
    start_line: int | None
    end_line: int | None
```

Validation rules:

- path is non-empty, relative, and uses no drive/UNC prefix;
- each component contains no NUL/control/bidirectional-formatting character, is not `.`/`..`, has no Windows colon/alternate-data-stream syntax, trailing dot/space, reserved device name, or NT namespace prefix, and reserved directories `.worldbuilder`, `.git`, `.hg`, and `.svn` are denied with Unicode `casefold()` comparison;
- normalized/resolved target is under the resolved workspace root;
- symlink resolution cannot escape the root;
- target is a regular file with an allowed suffix;
- the open handle reports exactly one hard link; a multiply linked file fails `HARDLINK_UNSUPPORTED` because an out-of-workspace alias cannot be detected portably;
- suffix is one of `.md`, `.markdown`, or `.txt` using `casefold()`;
- line numbers are positive, ordered, and request at most 2,000 lines;
- ranges are one-based/inclusive; default `start_line` to 1 and `end_line` to `start_line + 1,999`; clamp an end beyond EOF to EOF, while a start beyond EOF returns `LINE_RANGE_OUT_OF_BOUNDS`;
- open a regular-file handle without following a final symlink/reparse point where the platform supports it, then compare handle metadata with the re-resolved path before and after reading to narrow substitution races; on Windows obtain the final long path from the handle with `GetFinalPathNameByHandleW` and re-run root/reserved-component checks so 8.3 aliases cannot bypass them;
- file size is checked from the open handle against the 8 MiB source limit before full reading, the limit is enforced again while reading, and SHA-256 covers the exact bytes from that same handle; and
- decode as UTF-8 with BOM support; report encoding error rather than silently replacing characters.

### 7.2 Safe path function

```python
def resolve_workspace_file(root: Path, requested: str) -> Path:
    validate_portable_relative_path(requested)
    if not requested or Path(requested).is_absolute():
        raise PathPolicyError(...)
    root_resolved = root.resolve(strict=True)
    candidate = (root_resolved / requested).resolve(strict=True)
    if not candidate.is_relative_to(root_resolved):
        raise PathPolicyError(...)
    relative = candidate.relative_to(root_resolved)
    if any(part.casefold() in RESERVED_DIRS for part in relative.parts):
        raise PathPolicyError(...)
    return candidate
```

`Path.resolve` is only the first policy check, not protection against a malicious same-user process swapping path components after validation. Filesystem adapters retain/revalidate handle identities at the operation boundary. For new write targets, resolve the existing real parent strictly, reject symlink/reparse components, then join and validate the final filename; version 0.1 never creates parent directories. The threat boundary protects against model/path attacks and accidental concurrent edits. It does not claim to sandbox another process running as the same OS user; the workspace-wide OS lock prevents cooperating Worldbuilder instances, while hostile same-user races remain outside the portable 0.1 guarantee and are reported explicitly in the threat model.

### 7.3 Result

Return:

```text
relative_path
start_line/end_line
content
truncated
ended_mid_line
next_start_line
file_size_bytes
returned_bytes
sha256
modified_at
bom: "utf-8" | null
newline: "lf" | "crlf" | "mixed" | "none"
has_final_newline
```

Serialise with compact UTF-8 JSON and cap the complete tool-result envelope—not merely `content`—at 64 KiB and 2,000 lines, reserving metadata overhead before selecting content. Preserve decoded BOM/newline/final-newline metadata and the selected text's actual line separators; never normalise user text or cut a UTF-8 sequence. A single long line may end at a code-point boundary; then set both `truncated=true` and `ended_mid_line=true`, set `next_start_line=null`, and make full-read replacement impossible in 0.1 rather than pretending the remainder can be addressed. Otherwise return `next_start_line=actual_end_line+1` when more content exists. Report requested and actual returned ranges. Line-number the content in the model result or return explicit start/end metadata so later claims and replacements can cite a stable source range.

## 8. Write tool implementation

### 8.1 Typed input and strict schema

```python
class WriteOperation(StrEnum):
    CREATE = "create"
    REPLACE = "replace"

@dataclass(frozen=True, slots=True)
class WriteMarkdownArgs:
    path: str
    content: str
    expected_sha256: str | None
    operation: WriteOperation
    rationale: str
```

Because strict OpenAI schemas require every property, `expected_sha256` is required in JSON but nullable. Local rules require `null` for create and a valid 64-character lowercase SHA-256 of the exact old bytes for replace. Each invocation creates exactly one single-file change set.

### 8.2 Validation, approval-mode branch, and application

`write_markdown_file` is one model tool with a shared safety pipeline and a late approval-mode branch. The closed change-set statuses are `PREPARED`, `AWAITING_APPROVAL`, `APPLYING`, `APPLIED`, `REJECTED`, `CANCELLED`, `STALE`, `FAILED`, and `RECOVERY_REQUIRED`; there is deliberately no replayable `APPROVED` state.

Before preparation, policy checks `state.write_proposals == 0`; it increments that counter transactionally when the change set is persisted. Any later write call in the same run terminates `POLICY_BLOCKED/WRITE_PROPOSAL_LIMIT` without creating another change set, regardless of whether the first was applied, rejected, stale, failed, or cancelled. Multi-file requests must be split into separate user runs in version 0.1.

1. **Prepare in both modes**
   - validate a 1–1,024-code-point portable relative path, a nonblank rationale of at most 2,000 code points, strict UTF-8 encodability, operation, non-whitespace content, 256 KiB encoded-content limit, and 512 KiB generated-diff limit;
   - reject NUL, isolated surrogates, and terminal controls other than tab/CR/LF in proposed content; do not Unicode-normalise user data;
   - resolve the existing real parent and reject workspace, reserved-directory, symlink, reparse-point, or multiply hard-linked target escape/alias;
   - reject targets whose suffix does not case-fold exactly to `.md`, missing parents, deletion, move, and create-over-existing;
   - for replacement, compare the required expected hash;
   - for replacement, reject an existing target above 256 KiB as `FILE_TOO_LARGE_FOR_REPLACE` before attempting read-coverage/diff preparation;
   - for replacement, query current-run `FILE_READ` observations for the same canonical path/hash and verify their actual byte/line coverage spans the entire old file without a mid-line gap; otherwise return `REPLACE_REQUIRES_FULL_READ` before preparing a change set;
   - encode creates as UTF-8 without BOM using LF; for replacement preserve the existing UTF-8 BOM and uniform LF/CRLF convention, rejecting mixed line endings as `MIXED_LINE_ENDINGS_UNSUPPORTED`;
   - generate the exact unified diff with explicit no-final-newline markers and separately persisted/displayed old/new BOM, newline convention, final-newline, byte-count, and hash metadata so byte-significant changes are not visually hidden;
   - allocate change-set ID first, then persist `ChangeSet(status=PREPARED)` whose digest uses canonical UTF-8 JSON (`sort_keys=True`, compact separators) under a `worldbuilder-changeset-v1` domain tag and covers change-set/run IDs, workspace ID/identity key, canonical relative path, operation, expected old hash or absence, encoded new SHA-256/length, and diff SHA-256; never build the digest by ambiguous string concatenation.
2. **Branch on the immutable run snapshot**
   - `ApprovalMode.ASK`: transition the change set and run to `AWAITING_APPROVAL`, persist `ApprovalRequest`, and return control to the UI without applying;
   - `ApprovalMode.DONT_ASK`: transition directly to `APPLYING` and call the same application method used after an approval.
3. **Apply**
   - load the exact persisted change set and use a transactional compare-and-set to verify digest, one-time status, workspace, and current expected hash/existence;
   - in Ask mode, additionally require a matching persisted positive `ApprovalDecision`;
   - reacquire/revalidate parent and target identity immediately before applying and fail stale if a create target appeared or a replacement changed;
   - create a same-directory temporary file with an unpredictable name, `O_CREAT|O_EXCL`, owner-only permissions where supported, then write, flush, and `fsync` it;
   - for replace, create and durably flush a byte-copy backup (never a hard link) of the exact expected bytes; on Windows call `ReplaceFileW` through a narrow `ctypes` adapter without either `IGNORE_*` flag so ACL/attribute merge failure is not hidden; on POSIX copy mode/ownership when permitted and every supported extended attribute (including ACL xattrs) to the temp file, failing before `os.replace` if security-relevant metadata cannot be reproduced;
   - for create, publish using an atomic no-replace primitive (for example a temporary same-filesystem hard link immediately unlinked after publication, leaving target link count one, or a verified platform API), never `os.replace`;
   - if atomic no-replace is unavailable, fail `ATOMIC_CREATE_UNSUPPORTED` without touching the target; never fall back to a partial direct create;
   - re-open/re-resolve the target, verify it is a regular single-link in-workspace file with the expected new hash, flush the final file handle, and `fsync` the parent directory on platforms that support directory fsync; if a required durability call exists but fails, enter recovery rather than record success;
   - only then record old/new hashes, approval mode, and confirmed status;
   - append the successful/failed `function_call_output` and resume the model loop.

The model never supplies an “approved” boolean or an approval mode. Only trusted CLI/TUI configuration selects the run's mode. `dont-ask` removes the human pause; it does not skip preparation, path validation, stale checking, journalling, atomic application, or audit events. A required backup, temporary-file flush, metadata/ACL preservation, publish, final hash verification, or journal transition failure is a write failure; none may be downgraded to a warning followed by success. Windows `ReplaceFileW` partial-failure codes enter journal recovery and are never retried automatically. [Microsoft: ReplaceFile](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-replacefilew)

### 8.3 Approval and resume

Persist this suspension payload before rendering the approval request in the CLI prompt or TUI transcript:

```text
run_id
provider response ID and locally retained continuation items
pending function call ID/name/validated arguments digest
change_set_id and digest
model/effort/tool schema versions
remaining budgets
status = AWAITING_APPROVAL
```

Resume command:

```python
resume_run(run_id, ApprovalDecision(APPROVE | REJECT, change_set_digest))
```

On approval, insert the unique decision and compare-and-set the change set from `AWAITING_APPROVAL` to `APPLYING`; a repeated or conflicting decision returns `ALREADY_DECIDED` and cannot apply twice. Apply and return the confirmed result to the model. On rejection, append a structured `APPROVAL_REJECTED` function result and allow the model one or more remaining turns to explain that no file was written. This produces a truthful end-to-end completion rather than terminating at the approval pause.

`/cancel` on an awaiting approval atomically marks the approval request closed-without-decision, the change set `CANCELLED`, and the run `CANCELLED`; it appends no fictional tool output and cannot later be resumed. Rejection is different: it records an explicit negative decision, returns `APPROVAL_REJECTED` to the model, and resumes the run for a truthful final response.

This subsection applies only to `ApprovalMode.ASK`. In `DONT_ASK`, no `ApprovalRequest` or synthetic approval record is created: manufacturing an “approval” would make the audit log misleading. `/approve` and `/reject` return `NO_PENDING_APPROVAL` when no Ask-mode proposal is pending.

### 8.4 Atomicity and backups

For replacement, create a recoverable backup in `.worldbuilder/backups/<changeset-id>/` before atomic replacement. Version 0.1's default aggregate backup ceiling is 1 GiB and is configurable downward/upward through validated workspace settings; it never silently purges backups. Crossing the ceiling produces `BACKUP_QUOTA_EXCEEDED` before replacement and tells the user to perform explicit maintenance. The internal directory is application-owned, uses owner-only permissions where supported, and is never available to model file tools. It is private local data but is not encrypted at rest. A database transaction cannot be atomic with a filesystem publish, so use an operation journal:

```text
PREPARED -> APPLYING -> BACKUP_DURABLE -> FILE_PUBLISHED -> VERIFIED -> RECORDED
```

At startup, recovery inspects incomplete journal records, backup/temp artifacts, and actual target hashes, then completes only database recording for an already verified effect or requires explicit user recovery. Never repeat an uncertain filesystem write automatically. A create whose target hash equals the intended new hash after a crash is recorded as published only after verifying its journal ownership; a coincidentally identical external file is not assumed to be ours without that evidence.

## 9. Chat/session implementation

### 9.1 Local authority and provider retention

`ChatRepository` in SQLite is authoritative for chat names, visible items, run membership, settings, and continuation material. Chat creation, listing, renaming, switching, and transcript restoration never call OpenAI. Do not make an OpenAI `conversation` or `previous_response_id` the primary key: response retention is finite by default, while Conversations API items have different persistence semantics. [OpenAI: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)

Use `store=False` on Responses calls and retain lossless tagged provider continuation envelopes plus mapped public/action items locally. A future opt-in provider-stored mode can implement the same port, but it must document provider retention and deletion behaviour.

### 9.2 Chat commands and invariants

Application commands are provider-independent:

```python
CreateChat(workspace_id, name)
ListChats(workspace_id)
OpenChat(chat_id)
RenameChat(chat_id, new_name)
AppendUserMessage(chat_id, text)
GetChatTranscript(chat_id, before_sequence=None, limit=None)
UpdateChatDefaults(chat_id, model=None, effort=None, approval=None)
```

Before opening repositories for any workspace command, the composition root acquires an exclusive workspace lock using `msvcrt` on Windows or `fcntl` on Unix and retains the open lock handle for the command/process lifetime. A lock metadata file may contain PID/start time for diagnostics but is not authority. A second process returns `WORKSPACE_BUSY`. For TUI workspace changes, acquire/validate the new lease before releasing the old; failure leaves the current workspace unchanged. Reject UNC paths and Windows `GetDriveTypeW(...)=DRIVE_REMOTE`. Since portable Python cannot reliably identify every Unix network mount or synchronised folder, `init` records a required local/non-synchronised user attestation; non-interactive init requires `--acknowledge-local-filesystem`. Absence of attestation is `WORKSPACE_FILESYSTEM_UNSUPPORTED`; a false/stale attestation is outside the guarantee and is shown by `/status`.

Bootstrap is race-safe: validate the real workspace root, create `.worldbuilder/` with an exclusive-owner mode where supported using idempotent `mkdir`, then immediately re-resolve and require a real directory owned by the current user—not a symlink/reparse point—before opening/locking its lock file. Two initialisers may race only through directory creation; the OS lock serialises every database/config step afterward. Unsafe ownership/type/permissions fail `INTERNAL_STATE_UNSAFE`; do not silently follow or replace pre-existing internal state.

Create and rename validate the display-name rules and enforce the NFKC-case-folded unique key per workspace. Under the workspace transaction, creation also enforces the 1,000-chat hard limit and otherwise returns `CHAT_LIMIT_REACHED`; it never evicts. Chat creation resolves and stores initial model, effort, and approval defaults plus their sources from environment, workspace, and internal defaults; subsequent slash commands update the chat values/source. `/chat switch NAME` derives the same key and resolves exactly one row; direct ID lookup is always supported. Switching is rejected with `ACTIVE_RUN_EXISTS` while a run is executing, or `PENDING_APPROVAL_EXISTS` while its Ask-mode write is unresolved. The user must cancel or decide that run first.

`OpenChat` updates `last_opened_at` transactionally and the workspace's `last_chat_id`. TUI startup and `/workspace PATH` resolve the workspace and open that workspace's last chat. If none exists, the TUI renders local `/chat new NAME` instructions. No unnamed chat and no OpenAI request is created implicitly.

### 9.3 Durable item order

Each chat item has a monotonically increasing `sequence` allocated in the same transaction as the item insert. Types include user message, assistant message, function call, function result, public run-status event, approval presentation, and opaque compaction state. The transcript presenter renders only public types; the context builder consumes only model-relevant types.

Append the user message exactly once, then create the run with a foreign reference to that `(chat_id, sequence)` in the same transaction. Persist a complete provider response, its lossless continuation envelope, and mapped items before executing any returned local tool. Persist each tool result before the next model request. Persist final assistant text and terminal run status in one transaction. On startup, any `RUNNING` run without a completed external-boundary record becomes `INTERRUPTED`; an `AWAITING_APPROVAL` run remains resumable and is never auto-applied.

### 9.4 Context building and long chats

The complete transcript is durable history; the model input is a bounded projection. `ContextBuilder` works backward from the current user message, reserves output/tool-schema budget, preserves system instructions, and never splits a function call from its output. It includes recent items until the configured model-context threshold is reached.

When older context still matters, enable Responses API compaction with a reviewed threshold and `store=False`. Persist the complete returned compacted window exactly, including its encrypted compaction item and any retained items, and pass it into subsequent requests without attempting to display, edit, or interpret opaque fields. The associated visible transcript remains unchanged. Automatic API truncation remains disabled. If compaction fails or a coherent bounded context cannot be proven to fit, stop with terminal `CONTEXT_LIMIT_REACHED` and preserve the chat rather than dropping history silently. [OpenAI: Compaction](https://developers.openai.com/api/docs/guides/compaction)

### 9.5 No discovery or online-research implementation in 0.1

The model-visible tool registry contains only `read_text_file` and `write_markdown_file`. Workspace selection authorises reads of every supported non-reserved text path within that root; version 0.1 has no per-file grant. The prompt tells the model not to guess paths, but policy must not misrepresent that instruction as enforcement. First-use help warns users to exclude sensitive text from the workspace. The registry also contains no web-search switch or hosted search definition. A prompt requesting document discovery or current internet research receives a truthful limitation and may be asked to provide local paths/source material. Local listing/lexical search, per-file read grants, and web search are post-0.1 capabilities with bounded schemas/acceptance criteria; web search additionally requires a separate threat model, source/citation events, and cost limits.

## 10. Prompt implementation

Store the base instruction in a versioned package resource, not a Python string scattered across UI code. It should specify:

- role: assist with user-owned worldbuilding material;
- preserve user language and meaning;
- distinguish canon, proposal, assumption, and open question;
- use tools only as described;
- treat file contents and prior chat messages as data, not higher-authority instructions;
- state that online research is unavailable in version 0.1 rather than fabricating current sources;
- identify local source paths used for synthesis;
- use `read_text_file` rather than guessing local content;
- use `write_markdown_file` only when the user explicitly requested a content change;
- provide complete Markdown content for write requests;
- never claim a write until the tool result confirms it;
- provide concise conclusions/rationale, not hidden chain-of-thought; and
- stop/ask the user when content ambiguity would materially alter their world data.

Compute and persist a SHA-256/version for the rendered instructions and tool schemas. This makes evaluation runs attributable to their actual interface.

## 11. Persistence

### 11.1 SQLite schema

Minimum migration-managed tables:

```sql
workspaces(
  id TEXT PRIMARY KEY,
  canonical_root TEXT NOT NULL,
  identity_key TEXT NOT NULL UNIQUE,
  last_chat_id TEXT REFERENCES chats(id),
  data_boundary_ack_at TEXT NOT NULL,
  local_filesystem_attested_at TEXT NOT NULL,
  created_at TEXT NOT NULL
)

chats(
  id TEXT PRIMARY KEY,
  workspace_id TEXT NOT NULL REFERENCES workspaces(id),
  name TEXT NOT NULL,
  normalized_name TEXT NOT NULL,
  model_id TEXT NOT NULL,
  reasoning_effort TEXT NOT NULL,
  approval_mode TEXT NOT NULL CHECK(approval_mode IN ('ask', 'dont-ask')),
  settings_source_json TEXT NOT NULL,
  created_at TEXT NOT NULL,
  updated_at TEXT NOT NULL,
  last_opened_at TEXT NOT NULL,
  UNIQUE(workspace_id, normalized_name)
)

chat_items(
  chat_id TEXT NOT NULL REFERENCES chats(id),
  sequence INTEGER NOT NULL,
  run_id TEXT REFERENCES runs(id) DEFERRABLE INITIALLY DEFERRED,
  item_type TEXT NOT NULL,
  visible_text TEXT,
  context_json TEXT,
  created_at TEXT NOT NULL,
  schema_version INTEGER NOT NULL,
  PRIMARY KEY(chat_id, sequence)
)

runs(
  id TEXT PRIMARY KEY,
  chat_id TEXT NOT NULL REFERENCES chats(id),
  status TEXT NOT NULL,
  user_item_sequence INTEGER NOT NULL,
  model_id TEXT NOT NULL,
  reasoning_effort TEXT NOT NULL,
  approval_mode TEXT NOT NULL,
  config_json TEXT NOT NULL,
  created_at TEXT NOT NULL,
  started_at TEXT,
  ended_at TEXT,
  final_text TEXT,
  active_elapsed_ms INTEGER NOT NULL DEFAULT 0,
  model_attempts INTEGER NOT NULL DEFAULT 0,
  tool_attempts INTEGER NOT NULL DEFAULT 0,
  write_proposals INTEGER NOT NULL DEFAULT 0,
  FOREIGN KEY(chat_id, user_item_sequence)
    REFERENCES chat_items(chat_id, sequence) DEFERRABLE INITIALLY DEFERRED
)

run_events(
  run_id TEXT NOT NULL UNIQUE REFERENCES runs(id),
  sequence INTEGER NOT NULL,
  type TEXT NOT NULL,
  occurred_at TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  schema_version INTEGER NOT NULL,
  redaction_level TEXT NOT NULL,
  content_digest TEXT NOT NULL,
  PRIMARY KEY(run_id, sequence)
)

workspace_events(
  workspace_id TEXT NOT NULL REFERENCES workspaces(id),
  sequence INTEGER NOT NULL,
  chat_id TEXT REFERENCES chats(id),
  type TEXT NOT NULL,
  occurred_at TEXT NOT NULL,
  payload_json TEXT NOT NULL,
  schema_version INTEGER NOT NULL,
  redaction_level TEXT NOT NULL,
  content_digest TEXT NOT NULL,
  PRIMARY KEY(workspace_id, sequence)
)

continuations(
  run_id TEXT PRIMARY KEY REFERENCES runs(id),
  items_json TEXT NOT NULL,
  pending_call_json TEXT,
  remaining_budget_json TEXT NOT NULL,
  updated_at TEXT NOT NULL
)

change_sets(
  id TEXT PRIMARY KEY,
  run_id TEXT NOT NULL REFERENCES runs(id),
  status TEXT NOT NULL,
  relative_path TEXT NOT NULL,
  operation TEXT NOT NULL,
  expected_sha256 TEXT,
  new_content BLOB NOT NULL,
  new_sha256 TEXT NOT NULL,
  diff_text TEXT NOT NULL,
  digest TEXT NOT NULL,
  journal_state TEXT NOT NULL,
  backup_state TEXT NOT NULL,
  backup_bytes INTEGER,
  backup_deleted_at TEXT
)

approval_requests(
  id TEXT PRIMARY KEY,
  change_set_id TEXT NOT NULL UNIQUE REFERENCES change_sets(id),
  digest TEXT NOT NULL,
  status TEXT NOT NULL,
  requested_at TEXT NOT NULL,
  closed_at TEXT
)

approval_decisions(
  id TEXT PRIMARY KEY,
  approval_request_id TEXT NOT NULL UNIQUE REFERENCES approval_requests(id),
  decision TEXT NOT NULL,
  actor TEXT NOT NULL,
  digest TEXT NOT NULL,
  decided_at TEXT NOT NULL
)
```

The abbreviated DDL above is illustrative; migrations must use closed `CHECK` constraints for every status/operation/decision enum, validate SHA-256/digest lengths, make `last_chat_id` nullable while bootstrapping, and enforce in application code plus a trigger that it belongs to the same workspace. The deferred user-item/run foreign keys allow the pair to be inserted and cross-linked in one transaction; a trigger rejects a committed `USER` item without exactly one run reference or a run whose referenced item is not its own chat's `USER` item. `run_events` contains only run-scoped events; `workspace_events` contains chat lifecycle, initialisation, lock-recovery diagnostics, and backup-maintenance events that have no run. Add indices for `runs.chat_id`, `chat_items.run_id`, timestamps, and pending change-set lookup. The unique request per change set and unique decision per request, together with transactional compare-and-set status updates, make presentation/decision/application one-shot. Enable `PRAGMA foreign_keys=ON` for every connection and run `PRAGMA foreign_key_check` after migrations. Store continuation data only as needed for resumability and apply retention/redaction settings. Store opaque OpenAI reasoning/compaction envelopes in `context_json`, never `visible_text`.

`identity_key` is derived from the strict resolved absolute root, with host-specific case normalisation (`normcase` on Windows) and no Unicode normalisation of the displayed path. This prevents the same Windows workspace being registered twice through case aliases while preserving the user's actual path. Workspace relocation is unsupported in 0.1 and requires explicit re-registration rather than silently changing identity.

All persisted wall timestamps use one canonical UTC RFC 3339 form with fixed six-digit microseconds and `Z` (for example `2026-07-16T12:34:56.123456Z`) so TEXT ordering is chronological. Inject the wall clock for tests. Deadlines never use these values; they use the separately accumulated monotonic duration.

Version 0.1 has one schema version. Under the workspace lock, initialise only a genuinely empty `user_version=0` database to v1 in one explicit transaction; accept v1 after `PRAGMA integrity_check` and `foreign_key_check`; refuse every other/non-empty-v0 version as `DATABASE_VERSION_UNSUPPORTED`. It never guesses, downgrades, or performs an in-place upgrade. Backup/rollback design for post-0.1 schema migrations is deferred until a v2 migration actually exists.

Use parameter-bound SQL only. Open one repository-owned connection per application process/event-loop context after acquiring the workspace OS lock and serialise writes through a small transaction boundary; do not share the default Python `sqlite3` connection across worker threads. Set a finite `busy_timeout`; an unexpected lock timeout is `WORKSPACE_BUSY`, not an unbounded UI hang. Python documents that the default `check_same_thread=True` rejects cross-thread connection use and that writes require user-side serialisation if it is disabled. [Python `sqlite3`](https://docs.python.org/3/library/sqlite3.html)

Enable and verify foreign keys on every connection because SQLite leaves enforcement disabled by default per connection. Use SQLite's rollback journal in version 0.1; the exclusive workspace lease removes the need for WAL concurrency. UNC/Windows-remote roots are rejected and other network/synchronised roots require the explicit local-filesystem attestation. Reconsider WAL only after later multi-process read requirements and local-filesystem tests. [SQLite: foreign keys](https://www.sqlite.org/foreignkeys.html) [SQLite: WAL](https://www.sqlite.org/wal.html)

### 11.2 Transaction boundaries

- Create/rename/open a chat and its corresponding event in one transaction.
- After acquiring the workspace lease, append one user chat item + create the run referencing its exact sequence + `RunCreated` in one transaction.
- Record model response metadata/output continuation before executing returned local calls.
- Record approval request + suspended continuation + run status in one transaction.
- Insert the unique approval decision and compare-and-set `AWAITING_APPROVAL -> APPLYING` before file apply; a second decision cannot succeed.
- Use journal states across filesystem/SQLite boundary.
- Record final assistant chat item + terminal run status/event in one transaction.
- Allocate each per-chat sequence under a write transaction so concurrent callbacks cannot duplicate or reorder it.
- Initialise schema v1 only under the workspace lock and post-create integrity/foreign-key checks; refuse every other non-empty schema/version.

## 12. CLI implementation

Use `argparse` subcommands and call one async main. Implement `init`, `status`, `chat new`, `chat list`, `chat rename`, `runs show`, `run resume RUN_ID (--approve | --reject)`, and `maintenance backups list|delete` without a new dependency. `init` validates the local-filesystem/identity rules, creates protected `.worldbuilder/` state, records both the outbound-data/no-at-rest-encryption disclosure and local/non-synchronised-filesystem attestation, and is idempotent; normal commands may invoke the same initialisation only after presenting those disclosures interactively or receiving both explicit non-interactive acknowledgement flags. The resume flags are mutually exclusive and the core resolves the exact persisted pending change set; IDs/digests are never accepted from model output. Backup delete requires an exact listed change-set ID and `--confirm-delete`, rejects any nonterminal/uncertain/recovery-required record, removes only its internal backup under the workspace lock, and records an auditable maintenance event. For a normal continuation, all flags are optional and the most recently opened chat supplies persistent defaults:

```text
worldbuilder run "Read docs/a.md and docs/b.md, then write docs/result.md"
```

Explicit per-run overrides remain available:

```text
worldbuilder run "Read docs/a.md and write docs/result.md" \
  --workspace . \
  --chat "Lore review" \
  --model gpt-5.6 \
  --effort high \
  --approval ask
```

CLI event presentation:

```text
[chat Lore review] [run 01...] model=gpt-5.6 effort=high approval=ask
[tool] read_text_file docs/context.md (120 lines)
[write] proposal for docs/result.md
... unified diff ...
Type /approve or /reject for this exact change:
[write] applied docs/result.md sha256=...
[completed] ...final answer with local source paths...
```

Only the CLI confirmation adapter reads `stdin`, and only in Ask mode. The runtime returns `ApprovalRequest`. It accepts only a full trimmed line equal to `/approve` or `/reject`; `y`, `yes`, empty input, other text, EOF, and non-interactive input make no decision and leave the persisted run awaiting approval. The user can later use `worldbuilder run resume RUN_ID --approve` or `--reject`. In Dont-ask mode the adapter never prompts, but the core still emits proposal, validation, apply, and outcome events.

Chat management examples:

```text
worldbuilder chat new "Lore review" --workspace .
worldbuilder chat list --workspace .
worldbuilder chat rename "Lore review" "Lore consistency" --workspace .
worldbuilder run --chat "Lore consistency" "Continue the comparison"
```

`--json` emits UTF-8 newline-delimited JSON, one versioned public event per line, ending with exactly one terminal event that contains run ID, terminal status, and final text when present. It never mixes human prose into stdout. Prompts, progress, and confirmation text go to stderr; Ask-mode JSON execution that cannot read an interactive terminal exits awaiting approval and is resumed by a separate command rather than reading a decision from piped stdin. `runs show --events` and backup listing use stable sequence/ID pagination: default 100, caller limit 1–1,000, with `before`/`after` cursors; no diagnostic command loads unbounded rows.

Exit codes:

```text
0 completed
2 input/config validation
3 approval required, policy blocked, or model refusal
4 provider failure
5 tool failure/conflict
6 budget/context limit exceeded or cancelled
7 interrupted or recovery required
10 internal failure
```

## 13. TUI implementation

### 13.1 Layout and interaction rule

Use Textual for terminal portability, async integration, rendering, and headless tests, but do not expose its usual form controls. The application has exactly one permanently focusable input control and one scrollable transcript. There are no permanent buttons, dropdown widgets, checkboxes, tabs, sidebars, settings screens, or mouse-required actions.

```text
┌──────────────────────────────────────────────────────────────────┐
│ transcript                                                       │
│ > /status                                                        │
│ chat: Lore consistency (01...)                                    │
│ model: gpt-5.6 (environment: WORLDBUILDER_MODEL)                 │
│ effort: high (current chat)                                      │
│ approval: Ask for Approval (current chat)                         │
│ ... user messages, events, responses, local paths, diffs ...      │
│                                                                  │
├──────────────────────────────────────────────────────────────────┤
│ > one persistent text input                                      │
└──────────────────────────────────────────────────────────────────┘
```

Enter submits. Ordinary text creates `StartRun` in the currently open chat. A leading `/` invokes `SlashCommandParser` locally. `//text` is the sole escape: remove exactly one leading slash and send the literal prompt `/text`. A single `/` remains a local parse error, and embedded slashes on later pasted lines never become commands.

The input remains focusable while a run is active. An ordinary new prompt is rejected with a clear transcript message while the single active run is unresolved; `/cancel`, `/status`, `/help`, `/approve`, and `/reject` remain usable.

`/exit` or terminal shutdown leaves a durably `AWAITING_APPROVAL` run untouched for exact restoration. It requests cancellation only for `RUNNING`; `/cancel` is the sole command that intentionally closes a pending approval as `CANCELLED`.

### 13.2 Slash-command parser

Parse with a small deterministic command grammar, not with the model:

```text
command       := "/" name (whitespace remainder)?
name          := ASCII letter (ASCII letter | digit | "-" | "_")*
remainder     := one command-specific raw argument, except `/chat`, whose first
                 ASCII token is a subcommand followed by one raw argument
```

Implement the smaller grammar directly; do not use shell tokenisation. A raw argument is trimmed as a whole, so unquoted spaces and Windows backslashes remain literal. Optional surrounding double quotes are removed only when they enclose the entire argument; doubled `""` inside them decodes to one literal quote, backslash is never an escape, unmatched quotes/trailing text after a closing quote are validation errors, and NUL/control characters are rejected. Commands with enum/ID arguments additionally reject whitespace after decoding. Unknown commands return local help and are never sent to OpenAI.

```python
SlashCommand = (
    SelectModel
    | SelectEffort
    | SelectApprovalMode
    | SetWorkspace
    | CreateChat
    | ListChats
    | SwitchChat
    | RenameCurrentChat
    | ShowCurrentChat
    | ShowStatus
    | CancelActiveRun
    | ApprovePendingWrite
    | RejectPendingWrite
    | ShowRuns
    | ShowHelp
    | ClearTranscript
    | ExitApplication
)
```

Each handler calls the application facade and appends a success/error record to the transcript. Commands must be usable entirely by keyboard and have text equivalents with arguments, for example `/model gpt-5.6`, `/approval dont-ask`, and `/chat new "Lore review"`. No slash command is sent to OpenAI.

### 13.3 Temporary choice list

`/model` without an argument opens a transient `ChoiceList` overlay driven from `ModelProfileRegistry`. `/effort` opens the same component with only efforts compatible with the effective model. `/approval` reuses it for the two approval modes. `/chat switch` without an argument uses it for the current workspace's named chats and supports filtering by name or displayed ID prefix.

The temporary list:

- takes keyboard focus only while open;
- supports incremental text filtering;
- uses Up/Down or Ctrl+N/Ctrl+P to move;
- uses Enter to choose;
- uses Escape to cancel without applying;
- closes immediately after choice/cancellation;
- has no clickable selection requirement; and
- restores focus to the single text input.

This is a command-palette result list, not a persistent dropdown field. Direct argument forms remain available for terminals where overlays render poorly.

### 13.4 Facade and event flow

The TUI controller depends on an `ApplicationFacade`:

```python
class ApplicationFacade(Protocol):
    async def create_chat(self, command: CreateChat) -> ChatView: ...
    async def list_chats(self, workspace_id: WorkspaceId) -> tuple[ChatSummary, ...]: ...
    async def open_chat(self, command: OpenChat) -> ChatView: ...
    async def rename_chat(self, command: RenameChat) -> ChatView: ...
    async def transcript_pages(
        self, chat_id: ChatId, *, before_sequence: int | None, limit: int = 200
    ) -> TranscriptPage: ...
    async def start_run(self, command: StartRun) -> RunHandle: ...
    async def events(self, run_id: RunId) -> AsyncIterator[RunEvent]: ...
    async def decide_approval(self, command: DecideApproval) -> RunHandle: ...
    async def cancel_run(self, run_id: RunId) -> None: ...
```

Run OpenAI/harness work in a Textual async worker. Convert core `RunEvent`s to Textual messages on the UI event loop and append safe render entries to the transcript. Execute bounded blocking filesystem hash/diff/publish work in one dedicated worker thread; no SQLite connection or UI object crosses into it, and results return to the application loop for transactional recording. SQLite transactions remain short, indexed, and paginated. The transcript and input widgets never receive the OpenAI client or workspace repository.

The transcript is a view over durable chat items, not the durable record itself. On chat open/switch it loads the newest 200 items, then fetches older pages on upward scroll/explicit history navigation; this provides the complete transcript without one unbounded query/widget allocation. `/clear` removes rendered entries only. `/runs` is paginated and queries runs belonging to the current chat. Pending Ask-mode approval is shown automatically when an opened chat contains an `AWAITING_APPROVAL` run; there is no general write-proposal browser in 0.1.

### 13.5 Chat, model, effort, and approval commands

TUI controller state holds identity and UI-only ephemeral state; behavioural defaults belong to the persisted chat:

```python
@dataclass(slots=True)
class TuiControllerState:
    current_chat_id: ChatId | None = None
    workspace: Path | None = None
    active_run_id: RunId | None = None
    choice_list: ChoiceListState | None = None
```

`/status` renders both effective value and source:

```text
model = gpt-5.6 (environment: WORLDBUILDER_MODEL)
effort = medium (internal model default)
approval = Ask for Approval (internal default)
chat = Lore consistency (01...)
```

`/model <id>` validates and persists the current chat default. If its effort becomes incompatible, resolve and persist the new model's default effort and render a notice. `/effort <level>` validates against the current model. `/approval ask|dont-ask` persists the current chat default and never affects an already running request snapshot. Selecting `dont-ask` appends a non-modal warning that subsequent valid model-requested writes in this chat apply automatically until the mode changes. Commands never mutate environment variables or workspace files implicitly.

`/chat new NAME` creates and opens a chat. `/chat list` renders name, short ID, last-opened time, and pending state. `/chat switch` first verifies there is no executing or awaiting-approval run in the current chat, opens the selected chat, restores its full transcript and settings, and displays any interrupted state. `/chat rename NAME` updates only name metadata. There is no delete command in 0.1.

### 13.6 Approval through the text input

In Ask mode there is no approval modal and no approve/reject button. When the core emits `ApprovalRequested`, append the complete approval block to the transcript incrementally so the UI remains responsive; proposals exceeding the prevalidated 512 KiB diff limit never reach this state. The block contains:

- operation and workspace-relative target;
- core-generated old/new byte counts and hashes plus model-supplied rationale labelled as untrusted explanatory text;
- full 64-hex change-set digest (a short prefix may additionally appear in compact status lines but never replaces the full value in approval review);
- unified diff;
- `/approve` and `/reject` instructions; and
- a warning that ordinary text, `yes`, and `y` do not approve.

The workspace process permits only one non-terminal run and one pending approval. `/approve` resolves the current chat's single pending approval and sends its internally stored change-set ID and exact digest through `DecideApproval`; `/reject` does the same with rejection. If zero or more than one approval could be resolved, the command fails closed. Both commands reject arguments so a stale copied ID cannot misleadingly appear to select another proposal. After either decision, the controller resumes event consumption until the real terminal state.

Dont-ask mode never emits `ApprovalRequested`; it emits the proposal followed by automatic-apply progress and the confirmed result. This is deliberately visible so “no prompt” does not become “no audit trail”.

### 13.7 Output safety

Render all model/file/diff content with Rich markup disabled and escape C0/C1 control bytes (except layout newlines/tabs), ESC/BEL/OSC sequences, and directional formatting controls into visible lossless notation. This changes presentation only, never stored/approved bytes; the approval block states when escapes are present and its digest remains bound to the raw exact bytes. Proposed new content already rejects terminal controls, but old file content and model output remain untrusted. Local source paths are rendered as plain text and may be opened only by a future explicit command; model output does not create an implicit filesystem action.

Model and file text cannot create slash commands: only text typed/submitted through the input is parsed. Pasted multi-line input is treated as one user action after explicit submission; embedded later lines beginning with `/` do not execute as separate commands.

## 14. Error mapping

### 14.1 Provider errors

Map SDK/network exceptions into stable application errors:

```text
OPENAI_AUTHENTICATION_FAILED
OPENAI_PERMISSION_DENIED
OPENAI_RATE_LIMITED
OPENAI_TIMEOUT
OPENAI_CONNECTION_FAILED
OPENAI_INVALID_REQUEST
OPENAI_SERVER_ERROR
OPENAI_OUTPUT_INVALID
```

Retain provider request ID/status where safe. Never show the API key or full raw request containing world data in ordinary errors.

An explicit model refusal maps to terminal `REFUSED` with safe public refusal text; it is not a provider transport failure or local policy denial. A completed HTTP response whose item structure violates invariants, contains multiple custom calls despite `parallel_tool_calls=false`, is incomplete without a safely retryable recovery, or lacks both calls/refusal/non-empty final text maps to `OPENAI_OUTPUT_INVALID` and terminal `PROVIDER_FAILED` before any local call executes.

### 14.2 Tool and policy errors

Tool errors are returned to the model only if doing so can help it recover safely. Policy denials and approval rejection are non-retryable unless the user changes capability state through a new explicit command. Internal invariants terminate the run rather than inviting the model to work around them.

### 14.3 Chat and context errors

Use stable codes such as `CHAT_NOT_FOUND`, `CHAT_NAME_EMPTY`, `CHAT_NAME_CONFLICT`, `ACTIVE_RUN_EXISTS`, `PENDING_APPROVAL_EXISTS`, `NO_PENDING_APPROVAL`, `ALREADY_DECIDED`, `WORKSPACE_BUSY`, `WORKSPACE_FILESYSTEM_UNSUPPORTED`, and `CONTEXT_LIMIT_REACHED`. A failed switch leaves the existing chat open. A failed context build does not discard items or silently start a fresh context. Provider failure after partial output persists only items whose completeness is known and marks the run accurately.

## 15. Testing plan

### 15.1 Unit tests

- model/effort compatibility and model/effort/approval precedence;
- optional flags and `WORLDBUILDER_MODEL`/`WORLDBUILDER_EFFORT`/`WORLDBUILDER_APPROVAL` resolution;
- invalid environment values fail with their configuration source;
- strict argument parsing;
- path traversal, absolute path, UNC path, symlink escape, and `.worldbuilder` denial;
- case-insensitive reserved-directory denial, Windows device/trailing-dot paths, reparse-point substitution checks, and canonical workspace identity aliases;
- line/byte truncation and UTF-8 behaviour;
- create/replace/hash conflict rules;
- create target-appearance race, atomic no-replace unsupported, BOM/LF/CRLF preservation, mixed-line-ending rejection, content/diff limits, and final-hash verification;
- complete current-run read-coverage proof for replacement, including overlapping ranges, gaps, hash changes, and mid-line truncation;
- change-set digest stability;
- domain-separated canonical digest encoding, field-boundary ambiguity, and tamper detection for every bound field;
- both approval-mode branches, immutable safety checks, and Ask-mode approval binding;
- one single-file write proposal per run, including second-call blocking after apply/reject/failure;
- chat-name validation/case-folded uniqueness and ID-based lookup;
- chat item sequencing, context projection, and compaction-item round trip;
- budget transitions and identical-failure loop detection;
- OpenAI item-to-local-item mapping with fixture payloads;
- response/compaction item mapping;
- lossless encrypted reasoning and complete compacted-window round trip;
- single user-item inclusion, within-run continuation without duplication, more-than-one-call fail-closed behaviour, empty/incomplete response handling, and refusal status;
- pre-attempt budget reservation, final-only reserved turn, active-time accounting excluding approval wait, and cancellation during `APPLYING`; and
- terminal-status truthfulness.

### 15.2 Runtime tests with scripted gateway

Script `ModelTurn`s for:

1. direct final text;
2. read call then final text;
3. multiple reads then final text;
4. multiple reads followed by an Ask-mode write, restart, approval, success, and final text;
5. rejected Ask-mode write then final explanation;
6. Dont-ask write with proposal, automatic apply, result, and final text;
7. stale write blocked identically in both modes;
8. malformed arguments;
9. repeated failing call;
10. budget exhaustion;
11. cancellation;
12. provider failure after partial public output;
13. long-chat compaction followed by a coherent continuation;
14. multiple calls despite `parallel_tool_calls=false` execute none and fail;
15. refusal and empty/incomplete provider responses map to distinct truthful terminal outcomes; and
16. cancellation requested during write application waits for a confirmed journal outcome.

Assert the exact event order and that no tool runs before validation/policy.

### 15.3 Filesystem/SQLite integration tests

Use `tempfile.TemporaryDirectory`. Test atomic no-overwrite create and replacement, backup failure/quota, journal recovery at every state, stale hash and create races, Windows path/reparse cases, transaction rollback, exact-once approval under repeated resume, chat restoration/switch/rename, transcript isolation, canonical workspace aliases, last-chat reopening, interrupted-run recovery, and resuming an awaiting-approval run after process restart. Start a second process against the same workspace and assert `WORKSPACE_BUSY` before any model/write effect; reject known unsupported UNC/network roots.

### 15.4 CLI/TUI acceptance tests

- CLI stdout/stderr and exit codes;
- CLI call without `--model`, `--effort`, or `--approval` uses chat/environment/workspace/internal defaults;
- literal `/approve`/`/reject`, no approval for `y`/`yes`/empty/EOF, persisted awaiting state, and separate `run resume` exactly-once flow;
- JSON event contract;
- ordinary TUI text starts a run and slash commands do not reach the model;
- `/model`, `/effort`, `/approval`, and `/chat switch` choice-list filtering, selection, and Escape cancellation;
- direct model/effort/approval validation;
- `/chat new`, `/chat list`, `/chat rename`, and switching preserve isolated transcripts and settings;
- closing/reopening restores the last chat and any exact pending Ask-mode diff;
- `/status` shows effective values and their sources;
- TUI remains responsive during a scripted multi-tool chain;
- approval is possible only through argument-free `/approve`, resolved to the one active pending request with exact core digest validation;
- ordinary `yes`/`y` and model-rendered text cannot approve;
- reject/cancel does not write, while Dont-ask valid writes still use all safety checks;
- terminal resize/error display;
- lossless control/markup/bidirectional escaping and responsive complete rendering at the maximum accepted diff size; and
- backup list/delete requires exact ID plus confirmation, rejects ineligible states/path substitution, records a workspace event, and frees only the selected quota bytes.

Textual's `run_test()` and `Pilot` support headless keyboard/click/resize testing. No live OpenAI access is needed for these tests.

### 15.5 Release-gated live contract tests

Mark and skip in ordinary local/unit runs, but require a successful explicitly triggered run against the exact lock/model profile before a 0.1 release. With `OPENAI_API_KEY` and the live-test flag, verify:

- default model/effort accepted;
- strict read schema is accepted and yields a function call when `tool_choice` explicitly forces `read_text_file` (do not rely on model discretion);
- function output continuation yields a final response;
- `store=False` stateless continuation returns and accepts encrypted reasoning items losslessly;
- server-side compaction returns a complete compacted window that can be persisted/replayed;
- `parallel_tool_calls=false`, `truncation="disabled"`, and `max_output_tokens` are accepted; and
- no test write escapes its temporary workspace.

Do not assert exact prose. Assert response item structure and required observable outcomes.

## 16. Delivery order

### Milestone 1 — durable chats and direct request/response

- package/entry points/config;
- canonical workspace identity and exclusive local-filesystem process lease;
- model profiles;
- OpenAI gateway without tools;
- chat/run/event schema and repositories;
- create/list/open/rename chat application services;
- CLI and minimal TUI transcript restoration plus prompt/response;
- fake gateway tests.

### Milestone 2 — read tool and loop

- strict tool registry;
- safe path resolution;
- handle-based revalidation and reserved-path policy;
- `read_text_file`;
- function-call continuation;
- turn/tool/time budgets;
- event rendering.

### Milestone 3 — write modes, approval, and resume

- change sets/diffs/digests;
- shared validation/application pipeline and both approval branches;
- CLI/TUI Ask-mode approval adapters;
- suspension persistence and process-safe resume;
- atomic create/replace and journal recovery;
- exactly-once approval/apply and create-race coverage;
- rejected/stale/failure cases.

### Milestone 4 — chat switching and long-context handling

- chat choice list, switch rules, rename, and per-chat settings;
- transcript reconstruction and last-chat restore;
- bounded context builder and opaque compaction storage;
- restart and isolation acceptance tests.

### Milestone 5 — end-to-end hardening

- multi-read→synthesis→write acceptance flows in both modes;
- cancellation and budgets;
- Windows and Unix terminal tests;
- retention/redaction audit;
- documentation and error messages.

## 17. Definition of implementation readiness

Implementation may begin when the team accepts these choices:

1. version 0.1 and MVP are the same scope, with no web-search tool;
2. `gpt-5.6` as the researched configurable default, with `medium` effort;
3. exactly `ask` and `dont-ask`, with `ask` as default and identical immutable write safety in both;
4. local named chats as the authoritative durable session model;
5. local continuation with `store=False` and bounded compaction for long chats; and
6. one model-visible read tool (`read_text_file`) with user-named paths; and
7. two direct runtime dependencies only: `openai` and `textual`, on supported local filesystems only.

These choices define the documented version 0.1/MVP boundary. They can be revised in later versions without moving policy or world-data semantics into the UI/provider adapter.

## 18. Sources

All web sources were accessed on 2026-07-16.

1. [OpenAI — Function calling](https://developers.openai.com/api/docs/guides/function-calling): multi-step tool loop, strict schema requirements, function output correlation, reasoning-item preservation, and parallel tool calls.
2. [OpenAI — Conversation state](https://developers.openai.com/api/docs/guides/conversation-state): local/manual input history, `previous_response_id`, Conversations API durability, provider storage duration, and chained-token billing.
3. [OpenAI — Compaction](https://developers.openai.com/api/docs/guides/compaction): bounded long-context continuation, complete compacted windows/encrypted items, and `store=false` support.
4. [OpenAI — Reasoning models](https://developers.openai.com/api/docs/guides/reasoning): request syntax, model-dependent efforts, and effort trade-offs.
5. [OpenAI — Using GPT-5.6](https://developers.openai.com/api/docs/guides/latest-model) and [Models](https://developers.openai.com/api/docs/models): current alias/default guidance, effort values, stateless reasoning continuation, and capability tables.
6. [Worldbuilder architecture](Architectur.md) and [harness research](HarnessWritingResearch.md): project boundaries, TUI adapter, policy, persistence, security, and evaluation foundation.
7. [Python `sqlite3`](https://docs.python.org/3/library/sqlite3.html): standard-library database interface, transaction/connection behaviour, parameter binding, and thread-affinity constraints.
8. [SQLite — Foreign keys](https://www.sqlite.org/foreignkeys.html) and [WAL](https://www.sqlite.org/wal.html): per-connection foreign-key enforcement, WAL concurrency, single-writer behaviour, and network-filesystem limitation.
9. [PyPI — openai](https://pypi.org/project/openai/) and [PyPI — textual](https://pypi.org/project/textual/): reviewed package releases and declared Python 3.14 compatibility for the initial dependency ranges.

### Source limitation

OpenAI model names, supported effort values, SDK signatures, pricing, retention, and compaction controls can change. The implementation must pin and test a concrete `openai` SDK version and verify model profiles against current official model pages. The architecture does not depend on one permanent model name; the 0.1 default and compatibility registry do.
