# Worldbuilder 0.1 (MVP) Requirements

**Status:** normative version 0.1 scope  
**Last researched:** 2026-07-16  
**Audience:** product, implementation, and test contributors  
**Related documents:** [Architectur.md](Architectur.md), [implementation.md](implementation.md), and [HarnessWritingResearch.md](HarnessWritingResearch.md)

## 1. Scope and terminology

In this project, **version 0.1** and **MVP** are exact synonyms. A requirement described as required for the MVP is required for version 0.1, and anything described as post-MVP is post-0.1.

Worldbuilder 0.1 is a local Python application that accepts a user's task, lets an OpenAI model answer directly or use a small controlled local tool set, and presents the result through both a scriptable CLI and an interactive TUI. It must support multi-step work such as:

> The user asks the model to inspect relevant local Markdown documents, compare and synthesise their contents, write the findings and conclusions to an explicitly named Markdown file, and report exactly what happened.

Version 0.1 is not a general autonomous computer agent. It is a bounded worldbuilding harness with explicit workspace scope, an OpenAI-only model adapter, durable named chats, a small local tool surface, and durable evidence of each run. **Web search and every other general network-research tool are outside version 0.1.** The only required network connection is the connection to the OpenAI API.

## 2. Version 0.1 requirements in spirit

Version 0.1 must feel like a focused text conversation with a capable, bounded agent rather than a configuration form. It must:

- accept non-empty, Unicode natural-language input through both CLI and TUI and preserve the user's language;
- send chat turns to OpenAI through the official Python SDK and Responses API;
- receive and interpret assistant text, local tool requests, usage, refusals, and errors;
- display public progress and the final response accurately, including the real terminal run status;
- stream public assistant/progress events by default on interactive CLI/TUI paths, with `--no-stream` selecting complete-response mode for CLI without changing semantics;
- expose a safe, workspace-scoped read tool for Markdown/text files;
- expose a safe, workspace-scoped Markdown write tool;
- offer exactly two user-selectable write-approval modes: `Ask for Approval` and `Dont ask for Approval`;
- in `Ask for Approval`, pause before each write is applied and require `/approve` or `/reject` after showing the exact diff;
- in `Dont ask for Approval`, automatically apply a write only after the same path, type, size, stale-content, and atomicity checks have succeeded;
- never let either mode enable world-document deletion, arbitrary filesystem access, shell execution, arbitrary HTTP access, Git integration, or model-controlled permission changes;
- perform no Git commands, staging, commits, branch operations, or other Git integration;
- allow model and reasoning effort to be selected explicitly while always providing usable defaults;
- read default model and effort from `WORLDBUILDER_MODEL` and `WORLDBUILDER_EFFORT` when set;
- fall back to internal model and effort defaults when neither explicit values nor environment defaults are present;
- keep model, effort, and approval flags optional for every normal CLI call;
- persist named chats locally so they survive application restarts;
- reopen and continue an existing chat without losing its visible transcript or model-relevant context;
- let the user create, list, rename, and switch between chats;
- associate each user turn and its tool chain with exactly one chat and one run;
- offer a text-first TUI with one persistent input field rather than buttons, permanent dropdowns, forms, or mouse-driven controls;
- interpret ordinary TUI text as a user message and text beginning with `/` as a local application command;
- open a temporary keyboard-operated selection list when a slash command requires choosing among valid options or chats;
- support longer chains in which the model reads multiple local documents, synthesises them, requests a write, optionally waits for approval, resumes, and reports the exact outcome; and
- retain observable chat turns, tool calls, policy decisions, configuration, and concise conclusions without requesting or storing private chain-of-thought.

“The model thinks about the material” means that the selected reasoning model may plan and synthesise at the configured effort and then provide a concise, user-visible rationale. It does not mean exposing hidden reasoning tokens.

## 3. User experience

### 3.1 Entry points

The installable application must expose at least:

```text
worldbuilder run [PROMPT] [OPTIONS]
worldbuilder run resume RUN_ID (--approve | --reject) [--workspace PATH]
worldbuilder tui [PATH]
worldbuilder init [PATH]
worldbuilder status [--workspace PATH]
worldbuilder chat new NAME [--workspace PATH]
worldbuilder chat list [--workspace PATH]
worldbuilder chat rename CHAT NAME [--workspace PATH]
worldbuilder runs show RUN_ID [--workspace PATH] [--events] [--limit N] [--before SEQUENCE]
worldbuilder maintenance backups list [--workspace PATH] [--limit N] [--after CHANGE_SET_ID]
worldbuilder maintenance backups delete CHANGE_SET_ID --confirm-delete [--workspace PATH]
```

`worldbuilder init` validates and creates the protected local `.worldbuilder/` state and records acknowledgement of both the OpenAI outbound-data/unencrypted-local-storage disclosure and the requirement that the root is local and not concurrently synchronised. It is idempotent. Interactive first use may run the same flow; non-interactive first use requires both `--acknowledge-data-boundary` and `--acknowledge-local-filesystem` and otherwise fails without creating a run. `worldbuilder run` is the scriptable CLI path. It starts a turn in the selected chat, or creates a named chat when explicitly requested by the corresponding option. If `PROMPT` is omitted in an interactive terminal, it may read one prompt from standard input. In a non-interactive environment, missing input is a validation error rather than an indefinite prompt. `worldbuilder run resume` is the only non-TUI way to decide and resume a persisted Ask-mode run; `--approve` and `--reject` are mutually exclusive, are bound internally to the persisted change-set ID/digest, and fail closed if the run is not awaiting exactly one approval.

Backup deletion is an explicit operational exception to the no-world-content-deletion rule. `maintenance backups delete` accepts only an exact locally listed change-set ID plus `--confirm-delete`, deletes only that change set's internal backup, records a maintenance event, and refuses active, pending, uncertain, or `RECOVERY_REQUIRED` change sets. It cannot be invoked by the model or TUI. No command deletes world Markdown in version 0.1.

`worldbuilder tui` opens the interactive Textual interface. On startup it reopens the most recently used chat for the selected workspace. If no chat exists, the TUI asks the user to create one with `/chat new NAME`; it must not silently send a prompt to OpenAI. CLI and TUI invoke the same application services and harness runtime.

### 3.2 Minimum CLI options

```text
--workspace PATH
--acknowledge-data-boundary          first-use/non-interactive init only
--acknowledge-local-filesystem       first-use/non-interactive init only
--chat CHAT_ID_OR_NAME
--new-chat NAME
--model MODEL_ID
--effort EFFORT
--approval ask|dont-ask
--max-turns N
--max-tool-calls N
--max-output-tokens N
--timeout-seconds N
--json
--no-stream
```

All flags are optional for an ordinary continuation of the most recently used chat. In particular, omitting `--model`, `--effort`, and `--approval` is the normal short form. The application resolves the effective settings and displays them before the first model request.

The product labels and canonical configuration values are:

| Product label | CLI/config value | Behaviour |
| --- | --- | --- |
| `Ask for Approval` | `ask` | Persist and show the exact write proposal, pause, and require `/approve` or `/reject` before application. |
| `Dont ask for Approval` | `dont-ask` | While effective for a chat/run, every model-requested write that passes the immutable checks is applied without another prompt and reported exactly. |

`Ask for Approval` is the internal default. `WORLDBUILDER_APPROVAL` may set the default to `ask` or `dont-ask`; an invalid value is a configuration error. In the TUI, `Dont ask for Approval` is a standing per-chat permission and remains active until the user changes it. A CLI `--approval` value overrides only that invocation unless an explicit chat-settings command is used. Every surface emits a prominent warning whenever the effective mode is Dont-ask, regardless of source; it states that any valid model-requested write—including one influenced by malicious workspace text—can apply without per-write confirmation and recommends Ask mode for untrusted corpora. `/status` keeps this risk and mode visible. Approval mode changes only whether a human confirmation pause occurs. It never changes the immutable workspace and tool safety boundaries described in section 4. In an interactive CLI Ask flow, only the literal full-line input `/approve` or `/reject` decides the displayed proposal; `y`, `yes`, an empty line, EOF, and non-interactive stdin do not approve and leave the run durably awaiting a later `run resume` decision.

### 3.3 Text-first TUI interaction

The TUI consists conceptually of:

1. a scrollable transcript/output area containing chat messages, public run events, assistant responses, diffs, errors, and status; and
2. one persistent text input at the bottom.

There are no permanent buttons, dropdown fields, checkboxes, toolbars, or settings forms. Pressing Enter submits the input. Input without a leading `/` starts a normal chat turn. Input beginning with `/` is parsed locally and is never sent to the model unless escaped by doubling the first slash: `//text` sends the literal prompt `/text`.

The version 0.1 slash-command surface is:

```text
/model [MODEL_ID]              show/select the model for the current chat
/effort [LEVEL]                show/select compatible reasoning effort
/approval [ask|dont-ask]       show/select the write-approval mode
/workspace [PATH]              show/change the workspace when no run is active
/chat new NAME                 create and open a named chat
/chat list                     list chats in the current workspace
/chat switch [CHAT]            switch chat; no argument opens a choice list
/chat rename NAME              rename the current chat
/chat current                  show the current chat name and ID
/status                        show chat, settings, and active-run state
/cancel                        request cancellation of the active run
/approve                       approve the one pending write after diff review
/reject                        reject the one pending write
/runs                          show recent runs for the current chat
/help                          show commands and input rules
/clear                         clear only the rendered view, not chat history
/exit                          close the TUI safely
```

Entering `/model`, `/effort`, or `/approval` without an argument opens a transient keyboard-operated choice list over or adjacent to the transcript. `/chat switch` without an argument uses the same interaction for available chats. Typing filters the list, arrow keys move selection, Enter chooses, and Escape closes without change. This is not a permanent dropdown control and does not require a mouse.

Direct command arguments apply after local validation. `/status` reveals the effective model, effort, approval mode, setting sources, current workspace, chat name/ID, budgets, and active/pending state. Model, effort, and approval selection are persisted as defaults of the current chat so switching away and back restores its behaviour. Every run also records its immutable effective snapshot.

In `Ask for Approval`, a pending write displays the exact path, operation, full digest, old/new byte counts and hashes, BOM/newline/final-newline metadata, and unified diff including no-final-newline markers. Because version 0.1 permits only one non-terminal run in a workspace process and one pending approval for that run, `/approve` and `/reject` intentionally take no ID. The adapter resolves the current chat's single pending request while the core validates its exact persisted change-set ID, digest, status, and one-time decision. Zero or ambiguous pending requests fail closed. Ordinary text such as `yes` or `y` never grants approval.

In `Dont ask for Approval`, no approval request is created. The core applies the change only after all write validations pass, emits a visible `WriteApplied` or `WriteFailed` event, and returns the result to the model. `/approve` and `/reject` then fail with `NO_PENDING_APPROVAL`.

### 3.4 Chat lifecycle and naming

A **chat** is the durable user-visible conversation. A **run** is one submitted user message plus all model turns and tool calls required to answer it. One chat contains an ordered sequence of runs and local application events.

Version 0.1 rules:

- every chat has a stable opaque ID and a non-empty display name;
- a workspace may contain at most 1,000 chats in version 0.1; creating another fails `CHAT_LIMIT_REACHED` without deleting anything;
- display names preserve the submitted Unicode text after trimming Unicode whitespace, contain 1–120 Unicode code points, contain no control or bidirectional-formatting characters, and are unique within one workspace by `NFKC(name).casefold()`;
- `/chat new NAME` creates and immediately opens a chat;
- `/chat rename NAME` changes only the display name, never its ID or history;
- `/chat switch` is allowed only when no run is actively executing; an awaiting-approval run must first be approved, rejected, or cancelled;
- changing `/workspace` opens that workspace's most recently used chat, or shows the local `/chat new NAME` instruction if it has none;
- closing the application never deletes a chat;
- the most recently opened chat per workspace is restored on the next TUI start;
- the complete visible transcript, tool outcomes, pending approval, effective per-chat settings, and context-continuation material required for future turns are stored before they are presented as durable;
- if the process exits while a run is waiting for approval, reopening that chat restores the exact pending diff without applying it;
- version 0.1 provides no chat deletion command; destructive lifecycle operations are deferred; and
- `/clear` clears only the current rendered view. Reloading or switching back reconstructs the transcript from durable storage.

Only one Worldbuilder process may hold a workspace's execution lease in version 0.1. Every CLI/TUI workspace command acquires an exclusive, OS-backed lock in `.worldbuilder/` before opening application repositories and holds it for the command/process lifetime; a second process fails promptly with `WORKSPACE_BUSY` and performs no model call or write. On TUI `/workspace`, acquire the new workspace before releasing the old one and leave the old workspace active if acquisition fails. Stale lock metadata is diagnostic only—the operating-system lock, not a PID file, is authoritative. UNC paths and Windows drives reported as remote are rejected. Because synchronisation/network mounts cannot be detected portably, initialisation also requires explicit user attestation that the root is local and not concurrently synchronised; otherwise it fails `WORKSPACE_FILESYSTEM_UNSUPPORTED`. The guarantee does not extend to a user falsely attesting or later moving/enabling sync for the root.

Local SQLite storage is authoritative. Version 0.1 must not rely solely on an OpenAI response or conversation identifier for chat continuity. OpenAI documents that response objects are retained for 30 days by default, while Conversations API objects and items persist without that 30-day TTL. Local authority gives the application explicit retention control and avoids making continued access to a chat depend on provider-side storage. Requests use `store=false` and reconstruct the necessary continuation items from the local chat record; failure to round-trip required stateless fields is a compatibility error, not permission to enable provider storage silently. [OpenAI: Conversation state](https://developers.openai.com/api/docs/guides/conversation-state)

Long chats require bounded model context even though their full visible history remains local. The context builder must select the most recent relevant items, preserve unresolved tool-call pairs, and use persisted server-side compaction when the model context approaches a configured threshold. OpenAI's Responses API supports server-side compaction with `store=false`; the returned compacted window can contain an encrypted compaction item plus retained items and must be stored and passed forward losslessly rather than interpreted as user-visible facts. [OpenAI: Compaction](https://developers.openai.com/api/docs/guides/compaction)

## 4. Functional requirements

### FR-01 — User input and chat turn creation

The application shall:

1. accept Unicode text from CLI and TUI;
2. reject empty or whitespace-only user messages;
3. reject messages above 256 KiB when encoded as strict UTF-8, NUL, or isolated surrogate code points before persistence/network use;
4. require or resolve one current named chat before submission;
5. persist the exact user message once as the run's referenced chat item before the first network call; version 0.1 diagnostic redaction must not remove chat content required for continuation;
6. preserve the user's language in the response unless another language is requested;
7. generate a run ID associated with the chat before the first network call; and
8. allow cancellation after submission.

### FR-02 — OpenAI request

The application shall call the Responses API through the official `openai` package. Each request contains:

- the selected model ID;
- the selected compatible `reasoning.effort` value;
- versioned application instructions;
- bounded continuation items built from the current chat;
- the available strict local function-tool definitions;
- `parallel_tool_calls=false`, automatic provider truncation disabled, and bounded output/configuration policy; and
- correlation metadata retained locally.

The request contains no web-search, browser, arbitrary HTTP, or other network-research tool. The API key comes from `OPENAI_API_KEY` or a future secret-store adapter and is never written to workspace files, prompts, SQLite event payloads, logs, or error messages. Before the first model run, the application discloses that user messages and any supported workspace text requested through `read_text_file` leave the machine for OpenAI, that 0.1 has workspace-wide rather than per-file read authority, and that sensitive text must be kept outside the root. It never proactively uploads the workspace wholesale or claims that `store=false` makes the request local.

The Responses API supports the required model response, reasoning configuration, and custom function-calling loop. OpenAI describes function calling as: supply tools, receive a call, execute application code, return the tool output, and receive a final response or further calls. [OpenAI: Function calling](https://developers.openai.com/api/docs/guides/function-calling)

### FR-03 — Response handling and display

The application shall distinguish at least:

- public assistant text;
- local `function_call` requests and outputs;
- reasoning-summary items if explicitly enabled and supplied by the API;
- refusal/safety response;
- incomplete/failed response;
- usage and provider request ID; and
- terminal completion.

The CLI and TUI show public output without private reasoning. When a run returns control, they display one state from this closed set; `AWAITING_APPROVAL` is the sole resumable non-terminal pause, and every other listed state is terminal:

```text
COMPLETED
AWAITING_APPROVAL
CANCELLED
BUDGET_EXCEEDED
CONTEXT_LIMIT_REACHED
POLICY_BLOCKED
REFUSED
PROVIDER_FAILED
TOOL_FAILED
FAILED
INTERRUPTED
```

A model sentence never overrides application state. If the model says a file was written but the write tool failed, the visible result says `TOOL_FAILED` and must not claim success. A refusal is `REFUSED`, not `COMPLETED` or `POLICY_BLOCKED`; a provider response with no tool call, no refusal, and no non-empty completed assistant text is `PROVIDER_FAILED` with `OPENAI_OUTPUT_INVALID`.

### FR-04 — Read tool

The model shall have this local function:

```text
read_text_file(path, start_line, end_line)
```

Required behaviour:

- `path` is workspace-relative;
- the resolved path remains inside the selected workspace;
- allowed extensions are exactly `.md`, `.markdown`, and `.txt`, compared case-insensitively;
- source files are capped at 8 MiB; the complete JSON tool-result envelope is capped at 64 KiB and 2,000 lines (content is shortened to leave metadata overhead) without splitting a UTF-8 code point, with `truncated=true`, `ended_mid_line`, and the actual returned range when more requested content exists;
- line ranges are one-based and inclusive; null `start_line` means 1, null `end_line` means `start_line + 1,999`, and an end beyond EOF returns through EOF without error;
- results include canonical relative path, requested/actual line range, `next_start_line` when continuation is line-safe, content, truncation/mid-line flags, size, and SHA-256 hash;
- binary, oversized, missing, escaping, reserved internal directories (`.worldbuilder`, `.git`, `.hg`, and `.svn`, compared case-insensitively), and unsupported paths produce structured errors;
- the tool never follows a symlink outside the workspace and rejects regular files with link count greater than one, because an external hard-link alias cannot be bounded portably; and
- tool results are treated as untrusted user data when returned to the model.

`read_text_file` is the only model-visible discovery/read tool in version 0.1. Selecting a workspace grants read authority to every supported, non-reserved text file inside it; there is no per-file allowlist in 0.1. Because the model has no list/search tool, the normal workflow supplies exact paths, and the prompt tells the model not to guess paths, but this is usability guidance rather than a security boundary. The first-use disclosure must tell users to keep sensitive Markdown/text outside the selected workspace. Document listing, lexical search, and per-file grants are deferred explicitly.

### FR-05 — Write tool and immutable permission boundary

The model shall have one narrow Markdown write function:

```text
write_markdown_file(path, content, expected_sha256, operation, rationale)
```

`operation` is `create` or `replace`. Deletion, move, arbitrary patch, shell execution, and writes outside the active workspace are not version 0.1 capabilities.

Every write, in both approval modes, must:

1. validate the strict tool schema and workspace-relative target whose suffix case-folds exactly to `.md`;
2. reject reserved internal directories, paths outside the root, symlink/reparse-point escapes, multiply hard-linked targets, missing parents, oversized content/diffs, unsupported operations, Windows alternate-data-stream/short-name aliases, and path components containing control/bidirectional-formatting or host-ambiguous names;
3. require and validate `expected_sha256` for replacement to prevent stale overwrite;
4. for replacement, require current-run read observations with that same hash whose actual ranges cover the complete existing file; otherwise return `REPLACE_REQUIRES_FULL_READ`;
5. generate and persist a change set and unified diff before changing the file;
6. bind the operation to the change-set/run IDs, exact workspace ID/identity key, canonical relative path, operation, old hash/absence, encoded new-byte hash/length, diff hash, and a domain-separated canonical change-set digest;
7. immediately before application, reacquire/revalidate the parent and target identity and recheck absence/hash to narrow time-of-check/time-of-use races;
8. create a same-directory temporary file exclusively, flush and `fsync` its bytes, preserve the replaced file's security-relevant metadata or fail, and publish without overwrite: replacement uses a verified platform atomic-replace primitive only after the expected hash still matches, while creation uses an atomic no-replace primitive and fails with `ATOMIC_CREATE_UNSUPPORTED` if the platform/filesystem cannot provide one;
9. verify the resulting target is a regular in-workspace file with the expected new hash before recording success;
10. record path, previous/new hash, byte count, approval mode, and confirmed outcome; and
11. return a structured write result to the model.

The approval-mode branch occurs only after preparation steps 1–6:

- `Ask for Approval`: persist `AWAITING_APPROVAL`, show the diff, and wait for the user. `/approve` applies that exact non-stale change once; `/reject` records rejection and leaves the file unchanged.
- `Dont ask for Approval`: persist the validated proposal and immediately attempt the same atomic apply. No approval token or human pause exists, but all other checks and audit events remain mandatory.

The model cannot call a policy-changing tool, cannot insert an approval field in tool arguments, and cannot interpret text from a document as permission. Only a CLI/TUI user command or trusted startup configuration can select the approval mode.

Version 0.1 permits at most one write proposal/change set per run and that change set contains exactly one file; a second write call is policy-blocked without preparation, and multi-file work requires separate user turns. Create requires `expected_sha256=null`, an existing real parent directory, and a still-absent target. Replace requires the SHA-256 of the exact old bytes and an existing target no larger than 256 KiB; larger files remain readable but fail `FILE_TOO_LARGE_FOR_REPLACE`. Paths contain 1–1,024 code points before platform validation; rationale is nonblank and at most 2,000 code points. Content must contain at least one non-whitespace code point—empty/whitespace-only create or replacement is denied as destructive erasure. Content is encoded strictly as UTF-8; create uses UTF-8 without BOM and LF line endings. Replacement preserves an existing UTF-8 BOM and a uniform existing LF/CRLF convention; mixed-line-ending files fail with `MIXED_LINE_ENDINGS_UNSUPPORTED` instead of being silently normalised. No Unicode normalisation is performed. NUL, isolated surrogate code points, and terminal-control characters other than tab/CR/LF are rejected from proposed content. The initial hard limits are 256 KiB for encoded new content and 512 KiB for the generated unified diff; a larger proposal is rejected before approval so the exact reviewed diff is never truncated.

### FR-06 — Model selection

Model selection precedence is:

1. explicit `--model` or persisted current-chat `/model` selection;
2. `WORLDBUILDER_MODEL`;
3. workspace configuration;
4. internal application default.

The local `ModelProfile` registry records model ID/alias, supported effort values, function-calling support, Responses API support, and suitability for the version 0.1 multi-tool workflow. Web-search support is irrelevant to 0.1 compatibility.

The currently researched default is `gpt-5.6`. The default is a central configuration value, not scattered literals. Model capabilities change, so the profile and default require an intentional documentation/test update. [OpenAI: Models](https://developers.openai.com/api/docs/models)

Unsupported models or capability combinations fail locally before an API call with `MODEL_UNSUPPORTED` or `MODEL_CAPABILITY_MISMATCH`.

### FR-07 — Reasoning-effort selection

The accepted effort values come from the selected model profile. Resolution order is explicit `--effort` or current-chat `/effort`, `WORLDBUILDER_EFFORT`, workspace configuration, then the selected model profile's internal default.

Initial policy:

- default to `medium` for the default model;
- recommend `low` for simple answers and short reads;
- recommend `medium` for normal worldbuilding and tool use;
- recommend `high` for long local-document synthesis and report generation;
- allow higher values only where the profile supports them and warn about latency/cost;
- reject unsupported combinations locally with `EFFORT_UNSUPPORTED`; and
- pass the value as `reasoning={"effort": selected_effort}`.

Effort controls model computation. It never changes filesystem scope, tool budgets, or approval mode. [OpenAI: Reasoning models](https://developers.openai.com/api/docs/guides/reasoning)

### FR-08 — Long local multi-tool chains

The harness continues until the model produces a final response or reaches a real terminal condition. It must support:

```text
user message in a named chat
  -> model reads one or more local documents
  -> model compares and synthesises their contents
  -> model requests a Markdown create or replacement
  -> harness validates and persists the exact proposal
  -> Ask mode: user approves/rejects; Dont-ask mode: core applies immediately
  -> harness returns the exact tool result to the model
  -> model gives a final answer with source paths and exact write status
  -> all visible/context state is durable in the chat
```

For every local function call, the harness executes the application-side function and appends `function_call_output` with the matching `call_id`. It losslessly preserves all returned response output items required for the next call, including encrypted reasoning/compaction data returned in stateless `store=false` mode, without displaying or interpreting those opaque fields. OpenAI notes that reasoning items returned with tool calls must be passed back with tool outputs for reasoning models. [OpenAI: Function calling](https://developers.openai.com/api/docs/guides/function-calling) [OpenAI: Reasoning](https://developers.openai.com/api/docs/guides/reasoning#preserve-reasoning-across-calls)

Every Responses request sets `parallel_tool_calls=false`; version 0.1 therefore accepts at most one custom function call in a provider turn. If a response nevertheless contains more than one, the harness executes none of them, records `OPENAI_OUTPUT_INVALID`, and terminates `PROVIDER_FAILED`. Assistant text accompanying a tool call is persisted/rendered only as non-final progress; completion requires a later completed response with no tool call and non-empty assistant text. This fail-closed rule avoids undefined ordering, missing call outputs, and writes performed from a response that violated the requested contract.

An Ask-mode chain must survive an approval pause and process restart. The chat, run configuration, provider output items, pending call ID, change-set digest, observations, budgets, and ordered events are committed before `AWAITING_APPROVAL` is rendered.

Initial configurable limits:

| Limit | Initial default |
| --- | --- |
| Model turns per run | 12 |
| Local function calls per run | 10 (must be lower than the model-turn limit) |
| Maximum output/reasoning tokens per model response | 16,384 |
| Wall-clock runtime excluding approval wait | 5 minutes |
| Approval wait | no auto-decision; resumable after restart |
| Readable source / per-read output | 8 MiB / 64 KiB and 2,000 lines |
| Per-write content / unified diff | 256 KiB / 512 KiB |
| Sequential identical failed calls | terminate `TOOL_FAILED` on the second identical failure |

Budget counters reserve and persist an attempt before crossing a provider/tool boundary, so the configured maximum cannot be exceeded by one. The tool-call limit must be below the model-turn limit. When only one model turn remains after observations, the next request is final-only (`tool_choice="none"`); no new side effect may consume the turn required to report existing results. The wall-clock budget is accumulated active runtime using a monotonic clock and excludes only persisted `AWAITING_APPROVAL` time; each provider call and retry is capped by the smaller of its transport timeout and the remaining run deadline. Budget exhaustion returns `BUDGET_EXCEEDED` with a partial-result summary; it is never reported as completion.

### FR-09 — Configuration and effective settings

Configuration precedence is:

```text
explicit CLI option
  > persisted current-chat slash-command/default value
  > supported WORLDBUILDER_* environment value (when creating/initialising a chat)
  > workspace configuration
  > internal application default
```

Secrets are the exception: the API key comes only from the environment/secret adapter. `WORLDBUILDER_MODEL`, `WORLDBUILDER_EFFORT`, and `WORLDBUILDER_APPROVAL` are optional. A new chat resolves and persists its defaults plus original source; after that, its persisted defaults intentionally outrank later environment/workspace changes until the user changes the chat setting. An explicit CLI value always outranks the chat for that invocation. Invalid configured values fail with an actionable configuration error even if a higher-precedence value exists, preventing a latent unsafe setting. `/status` and CLI status output show effective values and sources, workspace, chat, budgets, and schema/prompt versions. Every run persists a non-secret immutable configuration snapshot.

### FR-10 — Chat/session persistence

SQLite shall durably store:

- chat ID, canonical workspace identity (including a case-normalised identity key on Windows), unique display name, creation/update/open timestamps;
- current per-chat model, effort, and approval defaults and their provenance;
- ordered user, assistant, local tool, compaction/context, and system-status items;
- run-to-chat association, the exact single user-item sequence that starts each run, and monotonically increasing item sequence;
- pending approval and continuation state;
- most recently opened chat per workspace; and
- enough lossless, tagged OpenAI continuation state plus mapped chat context to rebuild a bounded request without SDK objects or lossy item conversion.

Chat switching and restoration must not require an OpenAI API call. A chat remains inspectable if the API is unavailable. Writes and important state transitions use SQLite transactions so a visible message never claims persistence that did not occur. Database constraints and compare-and-set state transitions enforce at most one unresolved approval per change set and prevent double approval/application even after restart; the workspace execution lease prevents a second local process from racing those transitions.

### FR-11 — Run history and evidence

The local event store records run/chat IDs, timestamps, user task according to retention mode, model/effort/approval snapshot, available tools and budgets, provider request IDs and usage, local tool arguments/results subject to redaction, change proposals and decisions, file hashes, errors/retries, terminal state, and retained final output. It stores neither API keys nor private chain-of-thought.

## 5. Required end-to-end scenarios

### Scenario A — Direct answer and continuation

A user creates a named chat, submits a direct question, receives `COMPLETED`, closes the application, reopens it, sees the transcript, asks a follow-up, and the answer correctly uses the retained conversation context.

### Scenario B — Chat creation, rename, and switching

The user creates two uniquely named chats, switches between them through `/chat switch`, renames one, and observes that each transcript and settings remain isolated. Switching requires no provider call.

### Scenario C — Read and answer

The user asks about a named local Markdown file. The model calls `read_text_file`, receives bounded content and hash, answers from it, and identifies the local source path.

### Scenario D — Ask-mode create and resume

The user asks for a new document. The model reads relevant local files, synthesises content, requests `write_markdown_file`, and the core persists a diff. The user closes and reopens the app, sees the same pending proposal, enters `/approve`, the application writes atomically, and the model receives success before its final response.

### Scenario E — Ask-mode rejection

The user enters `/reject`; no file changes, the model receives `APPROVAL_REJECTED`, and the final response states that the proposal was not written.

### Scenario F — Dont-ask automatic write

The current chat uses `Dont ask for Approval`. A valid write proposal passes all immutable checks, is applied atomically without a prompt, is recorded, and is reported accurately. An escaping or stale proposal remains blocked even in this mode.

### Scenario G — Stale replacement

The target changes after it was read. Its expected hash no longer matches; both modes reject it with `CHANGESET_STALE`, no overwrite occurs, and the run ends `TOOL_FAILED` after a truthful explanation. Because 0.1 allows one write proposal per run, a fresh user turn must re-read and prepare a new proposal.

### Scenario H — Failure or budget exhaustion

The application preserves partial local results, reports `PROVIDER_FAILED`, `TOOL_FAILED`, or `BUDGET_EXCEEDED` accurately, and does not present an incomplete write as successful.

### Scenario I — Create race and exactly-once apply

A valid create proposal is prepared, but another file appears at the target before apply, or the same approval is submitted twice after restart. The application returns `CHANGESET_STALE`/`ALREADY_DECIDED`, does not overwrite the newly appeared file, and records at most one application outcome.

### Scenario J — Concurrent process and unsafe filesystem

While one process owns a workspace, a second CLI/TUI process fails with `WORKSPACE_BUSY` before a model request or mutation. A UNC/Windows-remote root or missing local/non-synchronised attestation fails validation with `WORKSPACE_FILESYSTEM_UNSUPPORTED`; it is never accepted silently with weaker locking or atomicity.

### Scenario K — Backup quota and explicit maintenance

A replacement that would cross the backup ceiling fails `BACKUP_QUOTA_EXCEEDED` before target mutation. `maintenance backups list` reports eligible exact change-set IDs/sizes; deletion without `--confirm-delete`, for an unknown/ineligible ID, or through a substituted internal path fails closed. Confirmed deletion removes only the selected eligible backup, records a workspace event, and a fresh user run can retry the replacement.

## 6. Non-functional requirements

### Safety and content integrity

- No path outside the selected workspace can be read or written.
- Reserved internal/VCS directories are denied case-insensitively, and every filesystem boundary revalidates against symlink/reparse-point substitution.
- No world-document deletion, arbitrary shell, arbitrary HTTP client, network-research, or executable-code tool is exposed; explicit ID-bound backup maintenance is not a model tool.
- The model cannot grant itself permission or change approval mode.
- Both approval modes use atomic, stale-safe writes and the same validation boundary.
- User content language and meaning are preserved.
- Proposed facts are not silently promoted to canon.
- SQLite, backups, journals, and temporary files contain private user data. They are created with owner-only permissions where the platform supports them, excluded from model tools, never advertised as encrypted at rest, and must fail closed if a required backup or durable flush cannot be completed.
- Replacement backups have a validated, configurable 1 GiB aggregate default ceiling; version 0.1 never purges them automatically, blocks the next replacement before mutation when the ceiling would be exceeded, and offers only explicit ID-bound CLI maintenance for eligible terminal backups.

### Responsiveness and durability

- CLI and TUI show chat/run identity and initial status promptly.
- Network/model work does not block TUI input handling.
- Cancellation remains available during tool loops.
- Streaming is presentation-only; correctness uses complete validated items.
- Provider auto-truncation is disabled; context overflow is reported as `CONTEXT_LIMIT_REACHED`, never handled by silently dropping an unknown prefix.
- Acknowledged chat items and pending approvals survive a normal application restart.

### Portability and dependencies

- Python `>=3.14`, subject to supported dependency releases.
- Only two direct runtime dependencies: `openai` and `textual`.
- Standard library for CLI, filesystem, SQLite, hashing, diffing, configuration, and tests initially.
- Write-capable workspaces require a supported local filesystem with verified locking, atomic replace, and atomic no-overwrite create semantics.

### Testability

- Most tests use a scripted fake model gateway and temporary workspaces.
- Network integration tests are optional and never required for ordinary unit tests.
- Every required scenario has automated acceptance coverage or a documented manual terminal test.

## 7. Explicitly deferred capabilities

- web search, online research, browser automation, and arbitrary network fetching/scraping;
- providers other than OpenAI;
- autonomous multi-agent delegation;
- world-document deletion and move tools;
- arbitrary writes outside the strict Markdown/workspace boundary;
- shell/code execution;
- background runs that continue after the local process exits;
- vector database/embedding search;
- model-visible document listing or lexical-search tools (users name paths in 0.1);
- web or desktop GUI;
- chat deletion, archive, export/import, sharing, and cloud synchronisation;
- Git status, diff, staging, commits, branches, or any other Git integration;
- exact replay of model reasoning; and
- requesting or displaying private chain-of-thought.

## 8. Version 0.1 completion definition

Version 0.1 (the MVP) is complete only when all required scenarios A–K pass through the shared core and applicable CLI/TUI surfaces, chats survive restart and can be switched safely, configuration compatibility is validated before API calls, both approval modes preserve the immutable safety boundary, concurrent-process/create/maintenance races fail closed, and a multi-read-to-Markdown chain reaches an accurately persisted and reported outcome.

## 9. Primary sources

All sources were accessed on 2026-07-16.

1. [OpenAI — Function calling](https://developers.openai.com/api/docs/guides/function-calling): function-call loop, strict schemas, tool outputs, reasoning/output-item preservation, and parallel-call behaviour.
2. [OpenAI — Conversation state](https://developers.openai.com/api/docs/guides/conversation-state): Responses continuation, Conversations API durability, and provider retention behaviour.
3. [OpenAI — Compaction](https://developers.openai.com/api/docs/guides/compaction): bounded long-conversation continuation and `store=false` support.
4. [OpenAI — Reasoning models](https://developers.openai.com/api/docs/guides/reasoning): `reasoning.effort`, model-dependent values, and latency/quality trade-offs.
5. [OpenAI — Using GPT-5.6](https://developers.openai.com/api/docs/guides/latest-model) and [Models](https://developers.openai.com/api/docs/models): current default alias, effort values, stateless encrypted-reasoning guidance, and model capabilities.
6. [Worldbuilder harness research](HarnessWritingResearch.md): harness responsibilities, context, policy, state, tool, security, and evaluation research.
7. [Python `sqlite3`](https://docs.python.org/3/library/sqlite3.html): standard-library local persistence and transaction/connection behaviour.
8. [SQLite — Foreign keys](https://www.sqlite.org/foreignkeys.html) and [WAL](https://www.sqlite.org/wal.html): referential-integrity enforcement and local-filesystem concurrency constraints.
