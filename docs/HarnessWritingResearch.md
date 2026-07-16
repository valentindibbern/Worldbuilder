# Research: AI / Agent Harnesses

**Research date:** 2026-07-16  
**Scope:** This is research on an *AI agent harness* (also called an *agent scaffold* or, in some sources, an *agent runtime*): the software system around a language model that lets it operate over multiple steps and affect an environment. It is not research on a software test harness. The document records terminology, mechanisms, documented behaviours, risks, and evaluation concepts; it deliberately contains no implementation plan or project-specific design.

## 1. Definition and the boundary of the term

Microsoft defines an agent harness as the “scaffolding” that turns a language model into an agent that can act. In that description, the harness runs the loop that calls the model and executes requested tools, manages conversation history/context, applies safety and approval policies before actions, and keeps the agent progressing until completion. The key distinction is that a model by itself produces an output, whereas a harness repeatedly interprets that output, performs authorised external work, returns observations to the model, and manages the session. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

Anthropic uses a compatible functional definition in its evaluation guidance: an agent harness (or scaffold) is the system that enables a model to act as an agent by processing inputs, orchestrating tool calls, and returning results. It treats the full multi-turn interaction as the relevant system under evaluation rather than treating a single model response as the complete agent. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

The term is not a formal, universally standardized architecture. Sources use it at different granularity: a small local tool-calling loop can be a harness; so can a long-running coding environment that includes state persistence, files, approvals, isolation, observability, and recovery. The stable common boundary is functional rather than brand- or framework-specific: the harness mediates between model output and the outside world across a task trajectory.

### 1.1 What an agent harness is not

- **A foundation model or chat-completions API:** these generate model output but do not themselves determine what external action follows, retain task state, or enforce application policy.
- **A prompt alone:** system instructions contribute to behaviour, but a harness also decides context selection, tool availability, execution, permissions, persistence, stopping, and reporting.
- **A tool server:** a tool server exposes capabilities; a harness chooses whether and when to expose, invoke, validate, and record them. Under MCP, servers expose tools, resources, and prompts, while the host/client is responsible for integrating them into the agent application. [MCP Specification: overview](https://modelcontextprotocol.io/specification/2025-06-18/server/index)
- **An agent framework or SDK:** a framework/SDK may supply reusable harness components or a ready-made harness. It is the package or platform; the harness is the runtime behaviour configured from those components.
- **An evaluation harness:** an evaluation harness runs measurements of an agent/harness. It is adjacent to, but distinct from, the production agent harness being measured.
- **An orchestration graph alone:** a graph can describe routing or task dependencies. A harness additionally owns the execution semantics around model calls, tools, state, policy, and terminal conditions.

## 2. The agentic execution loop

At its simplest, an agent harness repeatedly performs this sequence:

1. Receive a user task and current session/task state.
2. Assemble the model-visible context, including instructions, relevant state, tool definitions, and prior observations.
3. Invoke the model.
4. Interpret the response as either a final response or one or more requested actions.
5. Check requested actions against policy and approval state.
6. Execute permitted actions through controlled interfaces.
7. Convert action results, errors, and state changes into observations.
8. Persist or otherwise retain the relevant trajectory state.
9. Return to context assembly and repeat until a terminal condition is reached.

Microsoft explicitly describes automatic function invocation with a configurable iteration limit, history persistence after each model call, context compaction, approval/safety handling, and task progression as harness capabilities. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

This loop is often described as *reasoning and acting*, but a harness does not need access to private model reasoning in order to operate. It only needs an actionable model response in a defined interface: a final message, structured tool call, handoff, event, or failure. The harness’s control plane is therefore the sequence of model requests, model responses, allowed actions, action results, state transitions, and terminal outcomes.

### 2.1 Terminal conditions

An agent loop needs an explicit meaning for completion. Common terminal conditions include a model-declared final answer, a verified task result, user cancellation, iteration/token/time/cost budget exhaustion, policy refusal, unrecoverable tool failure, a required approval being denied or timing out, or an external cancellation signal. An iteration cap is not merely a performance control: Microsoft documents it as part of automatic tool invocation, making the cap an operational boundary on autonomous activity. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

A model’s declaration that it is finished is an observation, not independent evidence that the requested world change occurred. In multi-step agents, actual completion can be assessed through environment state, a tool result, a validator, or an evaluator. Anthropic’s agent-evaluation examples make this distinction concrete: an agent receives tools, a task, and an environment, runs a tool-calling loop, changes the environment, and is then assessed against the intended outcome. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

## 3. Major harness responsibilities

The following responsibilities recur in vendor documentation and agent-engineering literature. They are separable concerns; one runtime may combine them in a single process, while another separates them across services.

| Responsibility | Harness role | Evidence in sources |
| --- | --- | --- |
| Model invocation | Selects/configures a model call, supplies messages/tools, handles streamed or structured responses. | [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) |
| Context assembly | Chooses the instructions, history, retrieved information, tool schemas, and state exposed for the next turn. | [Anthropic: context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) |
| Action mediation | Parses requested calls; invokes tools; returns observations. | [Anthropic: agent evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) [MCP tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) |
| State and memory | Retains session/task state and provides recovery/continuity across turns or runs. | [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) |
| Policy and approvals | Determines whether an action is allowed, requires consent, is modified, or is blocked. | [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) [MCP specification](https://modelcontextprotocol.io/specification/2025-03-26/index) |
| Resource governance | Bounds turns, token use, wall time, concurrency, and tool execution resources. | [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) |
| Observability | Emits traceable records of model/tool/context events and their outcomes. | [OpenTelemetry GenAI](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/) |
| Evaluation and verification | Measures a final outcome and, where relevant, the multi-turn trajectory. | [Anthropic: agent evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) |

## 4. Context management

### 4.1 Context is a runtime resource

The context window is finite, whereas agent tasks can generate lengthy histories, tool outputs, files, retrieved documents, and intermediate state. The harness is the component that selects which information is supplied to each model invocation. In this sense, context management includes more than retaining chat history: it includes selection, ordering, summarization/compaction, truncation, retrieval, and the representation of durable task state.

Microsoft documents token-budget-aware compaction for long tool-calling loops. Its example requires a model context-window maximum and maximum output size to enable default compaction; it also permits custom strategies or disabling compaction. This shows that context handling is an explicit runtime behaviour with consequences for what an agent can remember and act on, not a transparent property of the model. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

Anthropic describes context engineering as the progression of prompt engineering for agents. Its documented emphasis is on managing the total information available to the model and on designing tools that are clearly understood and minimally overlapping. It notes that if a human engineer cannot decisively identify the appropriate tool in a situation, an agent cannot reasonably be expected to resolve that ambiguity more reliably. [Anthropic: Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)

### 4.2 Sources of context and their trust properties

Typical harness inputs include system/developer instructions, the current user request, previous messages, tool schemas, tool results, retrieved documents, workspace files, scratch/task state, summaries, and metadata such as identity or permissions. These inputs are not equivalent in authority or trustworthiness. A harness may receive untrusted natural language from users, web pages, documents, tool responses, or remote systems alongside higher-authority policy and operating instructions.

MCP formally distinguishes three server-provided primitives:

- **Prompts:** predefined templates or instructions for model interaction;
- **Resources:** structured data/content supplied as context; and
- **Tools:** model-controlled functions that allow actions.

The protocol’s specification also states that hosts must not transmit resource data elsewhere without user consent and calls out tool safety, user consent, and authorization flows as host responsibilities. [MCP Specification: server overview](https://modelcontextprotocol.io/specification/2025-06-18/server/index) [MCP Specification: general overview and security considerations](https://modelcontextprotocol.io/specification/2025-03-26/index)

### 4.3 Memory and persistence

“Memory” is not a single technical primitive. It can mean in-context conversation history; a persisted event log; externally stored task facts; files created by the agent; retrieval indexes; or application records. The harness determines lifecycle, access control, retention, and whether a memory item is reintroduced into the next context.

Microsoft’s harness persists chat history after every individual model call, which it identifies as enabling crash recovery and mid-run inspection. That design makes each call boundary a persistence point. It does not imply that all agent state or every external side effect is automatically transactional; external tools can have their own persistence and failure semantics. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

## 5. Tools and the action boundary

### 5.1 Tool schemas are part of the model interface

Tools turn model output into effects. A tool definition generally communicates a name, natural-language description, input schema, and sometimes result/error schema. Tool descriptions and schemas shape whether the model selects the intended capability and formats arguments correctly. Anthropic reports, from building its SWE-bench agent, that it spent more time optimizing tools than the overall prompt, and identifies tool design as a significant part of agent effectiveness. [Anthropic: Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents?via=aitoolhunt)

MCP defines tools as model-controlled functions exposed by servers. The protocol states that language models can discover and invoke them automatically from contextual understanding and user prompts. Tool results can include resource links, and content metadata can carry annotations such as audience, priority, and modification time. These mechanics mean that both tool definitions and tool outputs can enter the model-facing context. [MCP Specification: tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools)

### 5.2 Execution is not equivalent to model selection

The model can request a tool call; the harness decides whether to execute it and under which identity, scope, sandbox, timeout, retry policy, and audit record. This distinction is particularly important for side-effecting actions such as writing files, sending messages, changing records, making purchases, starting processes, or calling privileged APIs.

MCP’s specification places safety obligations on the host/client: it must build robust consent and authorization flows, and it must not send resource content elsewhere without user consent. The protocol supports authorization for HTTP transports, but authorization protocol support does not itself define a product’s business authorization policy or approval experience. [MCP Specification: overview and security](https://modelcontextprotocol.io/specification/2025-03-26/index) [MCP authorization specification](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2025-03-26/basic/authorization.mdx)

### 5.3 Tool failure and return semantics

Tools may return successful data, structured application errors, timeout/network failures, malformed data, access-denied responses, partial side effects, or opaque exceptions. A harness has to represent those outcomes to the model and to operators. A retry is not semantically neutral for a non-idempotent tool: repeating a request may duplicate a side effect. Likewise, a timeout does not reliably establish that the remote operation did not occur.

The agent loop can use tool failures as observations and adapt its next step. This adaptive property is why evaluating only the initial tool-selection decision is insufficient for multi-turn agents; the trajectory includes subsequent handling of errors and intermediate state. Anthropic’s evaluation guidance explicitly treats agents as systems that call tools, modify state, and adapt based on intermediate results. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

## 6. Policies, permissions, and approval gates

### 6.1 Policy belongs outside the model’s text response

A model instruction can influence behaviour, but a model-generated request is not an enforcement decision. In documented harness architectures, the control layer applies safety and approval policies before external actions. Microsoft lists tool approval, safety, and an iteration limit as explicit harness features rather than relying only on model instructions. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

MCP similarly makes hosts responsible for consent, authorization, and tool safety. Its “model-controlled” tool mechanism does not mean tools must be executed automatically or with the model’s full ambient authority. [MCP Specification: tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) [MCP Specification: overview](https://modelcontextprotocol.io/specification/2025-03-26/index)

### 6.2 Identity and delegated authority

Agent actions can involve multiple identities: the user, the application/tenant, the harness service, the tool server, and a delegated credential. The identity permitted to read a resource may differ from the identity permitted to write, send, approve, or delete. A harness records or conveys this operational identity at the action boundary; without it, an audit trail cannot reliably attribute an action to a user-approved delegation versus a broad service credential.

MCP’s HTTP authorization specification describes authorization at the transport level so that clients can request access to restricted servers on behalf of resource owners. It defines protocol mechanics for that delegation. It does not remove the requirement to map a tool action to the appropriate application-level permission and user-consent decision. [MCP authorization specification](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2025-03-26/basic/authorization.mdx)

### 6.3 Agentic security risks relevant to a harness

OWASP’s agentic-application security work identifies risks specific to autonomous, tool-using systems. Its published taxonomy includes agent goal hijack, tool misuse and exploitation, identity and privilege abuse, agentic supply-chain vulnerabilities, unexpected code execution, and memory/context poisoning. These are directly connected to harness responsibilities because the harness supplies context, exposes tools, handles identities, chooses execution environments, and persists memory. [OWASP: Top 10 for Agentic Applications](https://genai.owasp.org/2025/12/09/owasp-top-10-for-agentic-applications-the-benchmark-for-agentic-security-in-the-age-of-autonomous-ai/) [OWASP agentic applications document](https://genai.owasp.org/download/52117/?tmstv=1765059207)

The 2025 OWASP Top 10 for LLM Applications also identifies prompt injection, sensitive-information disclosure, supply-chain vulnerabilities, data/model poisoning, improper output handling, excessive agency, and system-prompt leakage. The “excessive agency” risk is especially relevant to agent harnesses because authority derives from the combination of tools, credentials, scope, and execution policy, not from the model alone. [OWASP: Top 10 for LLM Applications 2025](https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf)

## 7. Sandboxing and environment control

An agent does not act abstractly: a tool runs in an environment with a filesystem, network, processes, secrets, databases, browsers, cloud accounts, or other resources. The harness/environment boundary determines what the agent can observe or change and what happens if it acts incorrectly.

Anthropic’s evaluation guidance states that evaluating an agent requires running it in a real or sandboxed environment where it can use software applications and checking whether it reached the intended outcome. This establishes that environment construction is integral to realistic agent evaluation, not merely a deployment detail. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

Isolation has at least three meanings in this context:

- **Execution isolation:** processes, files, network access, and credentials are restricted to a defined surface.
- **Data isolation:** one user, tenant, task, or agent run cannot access another’s data without authorisation.
- **Evaluation isolation:** a run is placed in a controlled environment so the outcome can be attributed to the agent/harness rather than undocumented external variation.

The OWASP agentic taxonomy’s unexpected code execution, identity/privilege abuse, and supply-chain categories illustrate why execution environment and dependency provenance are security-relevant harness concerns. [OWASP: Top 10 for Agentic Applications](https://genai.owasp.org/2025/12/09/owasp-top-10-for-agentic-applications-the-benchmark-for-agentic-security-in-the-age-of-autonomous-ai/)

## 8. State transitions, recovery, and long-running work

Long-running work introduces failures that a one-shot chat interaction normally does not expose: process crashes, network interruptions, rate limits, expired credentials, tool-server restarts, partial actions, context-window exhaustion, user disconnection, and duplicate resumption. A harness needs a representation of what happened before the interruption if it is to continue or explain the run.

Microsoft’s documented per-service-call history persistence is specifically tied to crash recovery and mid-run inspection. Its compaction facility is tied to the finite context window of lengthy tool loops. These are examples of state-management mechanisms, not evidence that recovery is automatically safe for every external action: action idempotency, acknowledgement, and durable side-effect records remain tool/environment properties. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

Anthropic characterizes the Claude Agent SDK as a general-purpose agent harness for tasks requiring the model to use tools to gather context, plan, and execute, including long-running work. Its article on effective long-running harnesses is evidence that persistence and continuity are first-order concerns once a task spans extended time or multiple sessions. [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents?trk=article-ssr-frontend-pulse_little-text-block)

## 9. Observability and auditability

### 9.1 A trajectory is richer than a final answer

For an agent, the relevant execution record is a *trajectory*: a sequence of model calls, context inputs, tool calls, tool results, state changes, approvals/denials, errors, retries, and termination events. A final answer alone cannot reveal whether a tool was called, which data was read, whether an approval occurred, whether a failure was recovered from, or how much work/cost preceded the result.

Anthropic recommends preserving the full messages array at the end of an evaluation run, including all API calls and returned responses, for its API. It also describes evaluation systems that inspect more than a final text response: static analysis, browser-based checks, and LLM judging of behaviours such as instruction following. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### 9.2 Standardised telemetry

OpenTelemetry semantic conventions define common names and meanings for traces, metrics, logs, and events, enabling telemetry systems to interpret data consistently. OpenTelemetry has moved Generative AI attributes to a dedicated GenAI semantic-conventions repository; the conventions include attributes such as agent version. Its generative-AI material describes structured capture of model inputs, response metadata, and token usage. [OpenTelemetry: semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/) [OpenTelemetry: GenAI attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/) [OpenTelemetry: Generative AI](https://opentelemetry.io/blog/2024/otel-generative-ai/)

Microsoft states that its harness provides built-in OpenTelemetry observability following the generative-AI semantic conventions. This is an example of a harness exposing its internal events as operational telemetry rather than treating a model call as an opaque black box. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

### 9.3 Sensitive data in traces

Observability creates its own data-handling boundary. Trajectories can contain prompts, documents, source code, personal data, secrets, tool arguments, and tool results. A trace design therefore interacts with retention, redaction, access control, residency, and incident-response obligations. The need is not eliminated by structured telemetry; structured data can make sensitive information easier to query and correlate. MCP’s explicit requirement around user consent before transmitting resource data is one protocol-level example of this concern. [MCP Specification: overview](https://modelcontextprotocol.io/specification/2025-03-26/index)

## 10. Evaluation of an agent harness

### 10.1 Unit of evaluation

An agent evaluation can target different units:

- **Model response:** quality or correctness of one model output under fixed context.
- **Tool action:** selection of the appropriate tool and correctness of arguments.
- **Task outcome:** whether the required environment or user-visible result was achieved.
- **Trajectory:** whether the sequence of actions, policies, state transitions, and recovery was acceptable.
- **Harness configuration:** how a particular model, context policy, tool set, runtime, and approval configuration performs as a system.

Anthropic emphasizes that agent evaluation is harder because agents operate over many turns, call tools, modify state, and adapt. It identifies the agent harness as the system to run and evaluate, and describes evaluation in real/sandboxed environments with outcome checking. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### 10.2 Outcome versus trajectory

A successful terminal result does not necessarily imply that the route was safe, efficient, authorised, reproducible, or non-destructive. Conversely, different tool paths can sometimes reach the same valid result. These are why evaluation may separately measure final task completion and trajectory qualities such as correct tool use, policy compliance, resource use, recovery behaviour, and adherence to constraints.

LangChain’s agent-evaluation documentation defines agent evaluations as assessment of the execution trajectory—the sequence of messages and tool calls—not solely a final output. It also documents an LLM-as-judge trajectory evaluator. This is framework documentation rather than a universal standard, but it evidences a current engineering practice of treating the trajectory as a first-class evaluation object. [LangChain: Agent Evals](https://docs.langchain.com/oss/python/langchain/test/evals)

### 10.3 Evaluators and evidence

Agent-evaluation evidence can include deterministic checks (file contents, database state, compiler/test result, API record), task-specific validators, simulated environment outcomes, human review, and model-based judges. Each evaluator has a different error mode. For example, a deterministic parser can be precise about a schema but cannot judge subjective helpfulness; an LLM judge can assess nuanced behaviour but is itself a probabilistic model requiring calibration and explicit criteria.

Anthropic gives a concrete multi-method example: Descript’s agent evaluation system used static analysis, browser agents to test applications, and LLM judges for behaviours such as instruction following. It also reports dimensions for an editing workflow—avoid breaking things, perform the requested edit, and do it well—illustrating that outcome quality can have multiple independently assessed dimensions. [Anthropic: Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### 10.4 Regression and reproducibility

Model/provider versions, prompts, tool schemas, retrieval corpora, policy settings, dependencies, and environments can all change an agent’s behaviour. Evaluating a harness therefore involves identifying the configuration and environment that produced a run. A stored trajectory can aid inspection and replay, but exact replay can remain impossible when external systems, models, or nondeterministic tools change. This is a limitation of agent execution rather than a guarantee supplied by trace retention.

## 11. Reliability, budgets, and failure handling

### 11.1 Resource budgets

Agent loops consume resources across turns: input/output tokens, model latency, tool/API quota, compute, storage, network, human approval time, and potentially financial spend. A harness can account for these per call and across the task. Microsoft’s configurable iteration limit and token-aware compaction are direct examples of bounding loop growth and context consumption. [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

### 11.2 Failure categories

Agent-harness failures can arise at separate layers:

- model output is invalid, incomplete, or fails to produce an allowed action/final response;
- context is missing, stale, contradictory, overlong, or maliciously influenced;
- tool schema/selection/arguments are wrong;
- policy blocks an action, or an approval is unavailable/denied;
- tool execution fails, times out, or produces a partial external effect;
- state is lost or inconsistent during persistence/recovery;
- a sandbox, credential, network, or dependency fails;
- the agent exhausts a configured turn, time, token, or cost limit; or
- an evaluator or monitor cannot establish task completion.

Google’s and Anthropic’s sources both characterize agents as adaptive multi-turn systems that respond to intermediate tool observations. The implication for diagnosis is that a failure can originate before the final response, including at context construction, tool execution, or policy mediation. [Anthropic: Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents?via=aitoolhunt) [Anthropic: agent evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

### 11.3 Workflows, agents, and multi-agent systems

Anthropic distinguishes *workflows*—systems where predefined code paths orchestrate LLMs and tools—from *agents*, where the LLM dynamically directs its own process and tool use. It describes starting with an augmented LLM and increasing system complexity only through compositional patterns such as routing, parallelisation, orchestrator-workers, and evaluator-optimiser patterns. This is a taxonomy of agentic systems, not a claim that every harness must support each pattern. [Anthropic: Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents?via=aitoolhunt)

In multi-agent configurations, a harness may additionally define agent identities, delegation/handoff rules, shared or private state, tool/credential scopes, concurrency, aggregation, and termination. These controls expand the observable trajectory and the security boundary: tool use and information transfer may occur between agents as well as between a single agent and an external service.

## 12. Governance and human oversight

The NIST AI Risk Management Framework is voluntary guidance for incorporating trustworthiness considerations into the design, development, use, and evaluation of AI systems. The NIST AI RMF core includes governance outcomes that differentiate roles and responsibilities for human–AI configurations and AI-system oversight. This is a governance framework, not an agent-harness specification, but it supplies a relevant lens for agent systems whose runtime can initiate external actions. [NIST: AI RMF](https://www.nist.gov/itl/ai-risk-management-framework) [NIST AI RMF Core](https://airc.nist.gov/airmf-resources/airmf/5-sec-core/)

NIST’s Generative AI Profile applies the AI RMF to generative AI. NIST also reports that monitoring deployed AI is a broad, fragmented area and identifies challenges spanning system monitoring and human-system interaction measurement. These sources support treating monitoring, role definition, and oversight as lifecycle concerns rather than only model-training concerns. [NIST AI 600-1: Generative AI Profile](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf) [NIST: Challenges to monitoring deployed AI systems](https://www.nist.gov/news-events/news/2026/03/new-report-challenges-monitoring-deployed-ai-systems)

For an agent harness, human involvement can occur before a run (authorising scopes), during a run (approving selected actions or resolving ambiguity), or after a run (review, audit, or incident investigation). These are distinct operational roles. They should not be conflated with merely displaying an agent’s final natural-language response to a human after it has already acted.

## 13. Key distinctions for precise discussion

| Term | Precise meaning in this research | Relationship to an agent harness |
| --- | --- | --- |
| Model | A system that produces outputs from supplied inputs. | The harness invokes it; the model does not by itself own tool execution or policy. |
| Agent | A model-plus-runtime system that can pursue a task over multiple steps using observations/actions. | The harness supplies the runtime portion. |
| Harness/scaffold | The runtime/control layer that manages loop, context, tools, policy, state, and operational evidence. | The central subject of this document. |
| Tool | A defined external capability made available to the agent. | The harness exposes, mediates, executes, and records it. |
| MCP server | A protocol server exposing prompts/resources/tools. | A possible tool/context provider; not automatically the harness. |
| Memory | Retained session, task, or external knowledge/state. | The harness selects persistence and reintroduction into context. |
| Guardrail/policy | A rule or mechanism constraining outputs/actions. | The harness enforces action-time policy/approvals. |
| Trace/trajectory | Ordered evidence of the run’s calls, actions, observations, and transitions. | Produced/collected by harness observability. |
| Evaluation harness | A system for measuring agent/harness behaviour. | Distinct from the production runtime it evaluates. |

## 14. Findings and limitations of the evidence

1. **The term is operationally clear but not formally universal.** Microsoft and Anthropic independently converge on a runtime that processes input, invokes a model, orchestrates tools, manages context/state, and returns results. The exact component set varies by product and task. [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) [Anthropic](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents)

2. **The harness is a source of agent capability and constraint.** Tool availability, context selection, persistence, approval rules, stopping bounds, and environment access change what the same model can do. The model’s benchmark performance alone does not characterize the full agent system. [Anthropic: Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents?via=aitoolhunt) [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

3. **A tool request and an allowed action are different events.** MCP tool calls are model-controlled requests, but both MCP and Microsoft documentation assign hosts/harnesses responsibilities for consent, authorisation, safety, and approval. [MCP](https://modelcontextprotocol.io/specification/2025-03-26/index) [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness)

4. **Long-horizon operation makes state management and observability central.** Per-call persistence, context compaction, crash recovery, trajectory retention, and telemetry are documented harness features rather than optional afterthoughts in long-running-agent sources. [Microsoft](https://learn.microsoft.com/en-us/agent-framework/agents/harness) [Anthropic: long-running harnesses](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents?trk=article-ssr-frontend-pulse_little-text-block)

5. **Final-answer evaluation is incomplete for agents with external effects.** Agent evaluation sources explicitly assess task environments and trajectories. A terminal response cannot, on its own, establish correct tool use, policy compliance, absence of harmful side effects, or recovery quality. [Anthropic: agent evals](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) [LangChain: Agent Evals](https://docs.langchain.com/oss/python/langchain/test/evals)

6. **No source establishes deterministic replay as a general property of AI agents.** Persisted history and traces enable inspection and may support replay, but outputs, external tools, retrieval contents, remote services, and model versions can vary. Claims of exact reproduction must therefore be tied to the controlled model/environment configuration.

7. **This research does not equate agent capability with safety or policy compliance.** OWASP’s agentic risk taxonomy and NIST’s monitoring/governance work identify risks and oversight needs that arise precisely when an AI system gains tools, memory, and autonomous task progression. [OWASP](https://genai.owasp.org/2025/12/09/owasp-top-10-for-agentic-applications-the-benchmark-for-agentic-security-in-the-age-of-autonomous-ai/) [NIST](https://www.nist.gov/news-events/news/2026/03/new-report-challenges-monitoring-deployed-ai-systems)

## 15. Sources and methodology

The research prioritised primary or maintainer-authored material for concrete behaviour:

- [Microsoft Learn: Agent Harnesses](https://learn.microsoft.com/en-us/agent-framework/agents/harness) — explicit contemporary definition and documented harness features.
- [Anthropic: Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents?trk=article-ssr-frontend-pulse_little-text-block), [Building Effective AI Agents](https://www.anthropic.com/engineering/building-effective-agents?via=aitoolhunt), [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents), and [Demystifying evals for AI agents](https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents) — agent/harness, context, tool, long-running, and evaluation behaviour.
- [Model Context Protocol specification](https://modelcontextprotocol.io/specification/2025-03-26/index), [server overview](https://modelcontextprotocol.io/specification/2025-06-18/server/index), and [tools](https://modelcontextprotocol.io/specification/2025-06-18/server/tools) — protocol-level division of prompts, resources, tools, user consent, tool safety, and authorisation.
- [OpenTelemetry semantic conventions](https://opentelemetry.io/docs/concepts/semantic-conventions/) and [GenAI attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/) — interoperable telemetry terminology.
- [OWASP Top 10 for Agentic Applications](https://genai.owasp.org/2025/12/09/owasp-top-10-for-agentic-applications-the-benchmark-for-agentic-security-in-the-age-of-autonomous-ai/) and [OWASP Top 10 for LLM Applications 2025](https://owasp.org/www-project-top-10-for-large-language-model-applications/assets/PDF/OWASP-Top-10-for-LLMs-v2025.pdf) — agentic and LLM security-risk taxonomies.
- [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework), [Generative AI Profile](https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf), and [deployed-AI monitoring report announcement](https://www.nist.gov/news-events/news/2026/03/new-report-challenges-monitoring-deployed-ai-systems) — governance, oversight, and monitoring context.
- [LangChain Agent Evals](https://docs.langchain.com/oss/python/langchain/test/evals) — an example of trajectory-level evaluation support in a current agent framework.

All sources were accessed on 2026-07-16. “Harness” is an evolving industry term; product documentation describes specific implementations and defaults, not universal requirements. Security and governance sources identify risks and control domains, not proof that a particular product or harness is secure.
