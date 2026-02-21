#

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Natural Language Pipeline Authoring](#story-1-natural-language-pipeline-authoring)
    - [Story 2: Intelligent Pipeline Debugging](#story-2-intelligent-pipeline-debugging)
    - [Story 3: Pipeline Optimization Suggestions](#story-3-pipeline-optimization-suggestions)
    - [Story 4: Auto-Generated Documentation](#story-4-auto-generated-documentation)
    - [Story 5: Conversational Pipeline Exploration](#story-5-conversational-pipeline-exploration)
    - [Story 6: Agent-Driven Multi-Step Operations](#story-6-agent-driven-multi-step-operations)
    - [Story 7: External Tool Integration via MCP](#story-7-external-tool-integration-via-mcp)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Architecture Overview](#architecture-overview)
  - [LLM Integration Layer](#llm-integration-layer)
    - [Provider Abstraction](#provider-abstraction)
    - [Model Capability Types](#model-capability-types)
    - [Configuration](#configuration)
  - [Agentic Tool System](#agentic-tool-system)
    - [Tool Abstraction](#tool-abstraction)
    - [Built-in KFP Tools](#built-in-kfp-tools)
    - [Tool Approval and Security](#tool-approval-and-security)
  - [Chat Modes](#chat-modes)
  - [Automatic Context Gathering](#automatic-context-gathering)
  - [Prompt Rules System](#prompt-rules-system)
    - [Rule Definition Format](#rule-definition-format)
    - [Rule Storage and Discovery](#rule-storage-and-discovery)
    - [Rule Injection](#rule-injection)
  - [MCP Integration](#mcp-integration)
    - [KFP as MCP Server](#kfp-as-mcp-server)
    - [Consuming External MCP Servers](#consuming-external-mcp-servers)
  - [Backend Changes](#backend-changes)
    - [API Server Extensions](#api-server-extensions)
    - [Proto Definitions](#proto-definitions)
  - [Frontend Changes](#frontend-changes)
    - [AI Assistant Panel](#ai-assistant-panel)
    - [Rich Streaming Responses](#rich-streaming-responses)
    - [Inline AI Features](#inline-ai-features)
  - [Extension Point Architecture](#extension-point-architecture)
  - [SDK Extensions](#sdk-extensions)
  - [Security and Privacy](#security-and-privacy)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit Tests](#unit-tests)
    - [Integration Tests](#integration-tests)
    - [E2E Tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [Stable](#stable)
- [Frontend Considerations](#frontend-considerations)
- [KFP Local Considerations](#kfp-local-considerations)
- [Migration Strategy](#migration-strategy)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Fixed RPC-Per-Use-Case (No Agent Tools)](#fixed-rpc-per-use-case-no-agent-tools)
  - [SDK-Only Approach](#sdk-only-approach)
  - [Fine-Tuned Domain Model](#fine-tuned-domain-model)
<!-- /toc -->

## Summary

This proposal introduces Gen-AI capabilities into the Kubeflow Pipelines UI and backend, enabling
users to interact with their ML pipelines through natural language. The feature set includes natural
language pipeline authoring, intelligent debugging and log analysis, pipeline optimization
suggestions, auto-generated documentation, and conversational pipeline exploration.

The implementation adds an LLM integration layer to the API server with a pluggable provider
abstraction (supporting OpenAI-compatible APIs, Google Vertex AI, Amazon Bedrock, and self-hosted
models), an **agentic tool system** where the LLM dynamically invokes KFP operations during
conversations, **admin-configurable prompt rules** for injecting organization-specific guidelines,
and **MCP (Model Context Protocol) support** for interoperability with external AI tooling.

The architecture draws inspiration from
[Notebook Intelligence](https://github.com/notebook-intelligence/notebook-intelligence), adapting
its provider abstraction, chat modes, rules engine, MCP integration, and tool-calling patterns
to the Kubeflow Pipelines domain.

The feature is entirely opt-in, disabled by default, and requires explicit administrator
configuration to enable.

## Motivation

Kubeflow Pipelines is a powerful platform for building and deploying ML workflows, but it has a
steep learning curve. Users must understand the KFP SDK, pipeline DSL, Kubernetes concepts, and
Argo Workflows to effectively build, debug, and optimize pipelines. This creates friction for:

- **Data scientists** who are proficient in ML but not infrastructure or pipeline DSLs.
- **New adopters** who struggle to translate their ML workflows into KFP pipelines.
- **Operations teams** who need to quickly diagnose pipeline failures across hundreds of runs.
- **Organizations** that want to improve developer productivity and reduce time-to-production for
  ML models.

Gen-AI capabilities can significantly reduce this friction by allowing users to describe intent in
natural language and receive actionable pipeline code, debugging insights, and optimization
recommendations. Similar capabilities have proven transformative in adjacent tools (e.g., GitHub
Copilot for code, Jupyter AI for notebooks, Notebook Intelligence for JupyterLab).

### Goals

1. Enable users to describe pipeline logic in natural language and receive valid KFP Python SDK
   code that can be directly compiled and run.
2. Provide AI-powered analysis of pipeline run failures, including log summarization, root cause
   identification, and suggested fixes.
3. Offer pipeline optimization suggestions based on resource utilization, execution patterns, and
   best practices.
4. Auto-generate human-readable documentation for pipelines, components, and their parameters.
5. Support conversational exploration of pipeline history, run metadata, and experiment results.
6. Maintain a pluggable LLM provider abstraction with **separate model types per capability**
   (chat, embedding, inline completion) so organizations can use their preferred model provider
   and assign different models to different tasks.
7. Provide an **agentic tool system** where the LLM can dynamically invoke KFP operations
   (list runs, get logs, compile pipelines) during conversations, with user confirmation required
   for mutating actions.
8. Support **admin-configurable prompt rules** that inject organization-specific guidelines,
   best practices, and constraints into AI prompts without requiring model fine-tuning.
9. Integrate with the **Model Context Protocol (MCP)** ecosystem — both as an MCP server
   (exposing KFP operations to external AI tools) and as an MCP client (consuming external
   tools like model registries and data catalogs).
10. Expose an **extension point** for platform teams to register custom tools the AI assistant
    can invoke.
11. Ensure the feature is fully opt-in and disabled by default, with no impact on existing
    deployments.
12. Respect data privacy by allowing administrators to configure what context (logs, parameters,
    artifacts) is sent to the LLM provider.

### Non-Goals

1. Training or fine-tuning a Kubeflow-specific LLM. This proposal uses general-purpose LLMs with
   prompt engineering and retrieval-augmented generation (RAG).
2. Autonomous pipeline execution without user confirmation. The AI assists and suggests; the user
   always confirms mutating actions.
3. Replacing the existing SDK or UI authoring experience. Gen-AI capabilities augment the existing
   workflows.
4. Supporting real-time streaming inference pipelines. This proposal focuses on batch/training
   pipeline workflows.
5. Building a general-purpose chatbot. The AI assistant is scoped to Kubeflow Pipelines operations.
6. Full IDE-level inline code completion (this could be a future enhancement if an inline editor
   is added to the KFP UI).

## Proposal

Add a Gen-AI integration layer to the Kubeflow Pipelines backend and a corresponding AI assistant
interface in the frontend. Rather than exposing a fixed set of RPCs for predefined use cases, the
core design centers on an **agentic chat endpoint** where the LLM has access to a set of **KFP
tools** (list runs, get logs, get pipeline spec, compile pipeline, submit pipeline, etc.) and can
compose them dynamically to answer any question or perform any multi-step operation.

This agent-with-tools approach — inspired by Notebook Intelligence's `ChatParticipant` and tool
system — is fundamentally more flexible than static RPCs. Instead of needing a dedicated
`AnalyzeRun` endpoint, the LLM can autonomously call `get_run_status`, then `get_pod_logs`, then
`get_pipeline_spec` to build the context it needs to diagnose a failure. New capabilities emerge
from tool composition without new backend endpoints.

The feature is configured through the existing `pipeline-install-config` ConfigMap and is disabled
by default. When enabled, the API server exposes the agentic chat endpoint and optional convenience
RPCs that the frontend consumes.

### User Stories

#### Story 1: Natural Language Pipeline Authoring

As a data scientist, I want to describe my ML workflow in plain English (e.g., "Create a pipeline
that loads data from BigQuery, preprocesses it with pandas, trains an XGBoost model, evaluates it,
and deploys to a KServe endpoint if accuracy > 0.95") and receive valid KFP Python SDK code that I
can review, edit, and run.

#### Story 2: Intelligent Pipeline Debugging

As an ML engineer, I want to click on a failed pipeline run and ask "Why did this fail?" The AI
agent autonomously retrieves the run status, fetches pod logs for failed steps, reads the pipeline
spec, and presents a root cause analysis with suggested fixes — all without me specifying which
logs to pull.

#### Story 3: Pipeline Optimization Suggestions

As a platform engineer, I want to select a pipeline and ask "How can I optimize this?" The agent
retrieves recent run metrics, compares resource requests vs. actual usage, identifies cacheable
steps, and recommends parallelization opportunities — composing multiple tool calls to build a
complete picture.

#### Story 4: Auto-Generated Documentation

As a team lead, I want to select a pipeline version and click "Generate documentation" to produce
a human-readable description of what the pipeline does, its inputs/outputs, component
dependencies, and expected resource requirements, formatted as Markdown that can be stored
alongside the pipeline code.

#### Story 5: Conversational Pipeline Exploration

As a data scientist, I want to ask questions like "Show me all runs from last week that used the
v2 preprocessing component and had accuracy above 0.9" and receive a filtered view of matching
runs without needing to manually construct filters in the UI.

#### Story 6: Agent-Driven Multi-Step Operations

As an ML engineer, I want to say "Clone the production pipeline, update the training image to
v2.3, and submit a test run in the staging namespace" and have the agent break this into tool
calls (get pipeline spec, modify spec, compile, submit), showing me each step and asking for
confirmation before the mutating submit action.

#### Story 7: External Tool Integration via MCP

As a platform engineer, I want to connect our internal model registry as an MCP server so the
AI assistant can answer questions like "Which model version is deployed for the fraud detection
pipeline?" by querying both KFP and the registry in a single conversation.

### Notes/Constraints/Caveats

- **LLM availability**: The feature requires network access to an LLM endpoint (cloud API or
  self-hosted). Air-gapped environments must deploy a self-hosted model.
- **Cost**: Cloud LLM API calls incur per-token costs. The implementation should support token
  budgets and rate limiting to prevent unexpected charges.
- **Latency**: LLM responses may take several seconds. The UI must handle this gracefully with
  streaming responses and progress indicators.
- **Accuracy**: LLM-generated pipeline code may contain errors. The proposal emphasizes a
  human-in-the-loop approach where all generated code is presented for review, not auto-executed.
- **Context window**: Pipeline specs, logs, and metadata may exceed LLM context windows. The
  implementation must use intelligent context selection and summarization. Per-model
  `context_window` metadata (inspired by NBI's model properties) enables the context builder
  to adapt to each model's capacity.
- **Tool calling support**: Not all LLMs support tool/function calling. The provider abstraction
  exposes a `supports_tools` property per model so the system can fall back to prompt-based
  approaches for models without native tool support.

### Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| LLM generates incorrect pipeline code | Users run broken pipelines | Generated code is always presented for review; never auto-executed. Include a validation step that compiles the pipeline before presenting it. |
| Sensitive data sent to external LLM provider | Data leakage, compliance violations | Admin-configurable data filtering controls. Support self-hosted models. Redact secrets and credentials from context by default. |
| High cloud API costs from LLM usage | Unexpected billing | Configurable token budgets, rate limiting per user/namespace, and cost estimation displayed before expensive operations. |
| LLM provider outage | Feature unavailable | Graceful degradation — all existing UI/SDK functionality works without the AI features. Clear error messaging when AI is unavailable. |
| Prompt injection via pipeline names or parameters | Unauthorized LLM behavior | Sanitize all user-provided content before including in prompts. Use structured prompts with clear system/user boundaries. |
| Model hallucination of non-existent KFP APIs | User confusion | Include KFP SDK documentation as RAG context. Validate generated code against the installed SDK version. |
| Agent takes destructive action without consent | Data loss, unintended side effects | All mutating tools require explicit user confirmation (secured tool wrapping). Read-only tools are auto-approved. |
| Prompt rules contain conflicting instructions | Unpredictable AI behavior | Rules are priority-ordered; conflicts are resolved by highest priority. Validation warns on contradictions at load time. |
| External MCP servers introduce untrusted tools | Security boundary violations | MCP tools are wrapped with confirmation dialogs. Admins explicitly allowlist MCP servers. Tools from external servers are never auto-approved. |

## Design Details

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kubeflow Pipelines UI                        │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────┐  │
│  │ AI Assistant  │  │ Inline AI    │  │ Existing UI           │  │
│  │ Panel (Chat)  │  │ Actions      │  │ (unchanged)           │  │
│  │               │  │              │  │                       │  │
│  │ Rich Stream:  │  │ Contextual   │  │                       │  │
│  │ - Markdown    │  │ buttons on   │  │                       │  │
│  │ - Buttons     │  │ runs, pipes  │  │                       │  │
│  │ - Progress    │  │              │  │                       │  │
│  │ - Confirms    │  │              │  │                       │  │
│  └──────┬───────┘  └──────┬───────┘  └───────────────────────┘  │
└─────────┼──────────────────┼────────────────────────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                   KFP API Server                                │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              AI Service (new, optional)                     │ │
│  │  ┌────────────┐  ┌──────────┐  ┌────────────────────────┐ │ │
│  │  │ Prompt      │  │ Context  │  │ Response Processor     │ │ │
│  │  │ Templates   │  │ Builder  │  │ (rich stream types)    │ │ │
│  │  └────────────┘  └──────────┘  └────────────────────────┘ │ │
│  │  ┌────────────────────────────────────────────────────────┐│ │
│  │  │              Agentic Tool System                        ││ │
│  │  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  ││ │
│  │  │  │ KFP      │ │ MCP      │ │ Extension│ │ Secured  │  ││ │
│  │  │  │ Built-in │ │ External │ │ Custom   │ │ Wrapper  │  ││ │
│  │  │  │ Tools    │ │ Tools    │ │ Tools    │ │ (confirm)│  ││ │
│  │  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘  ││ │
│  │  └────────────────────────────────────────────────────────┘│ │
│  │  ┌────────────────────────────────────────────────────────┐│ │
│  │  │              Prompt Rules Engine                        ││ │
│  │  │  ┌───────────────┐ ┌───────────┐ ┌──────────────────┐ ││ │
│  │  │  │ Rule Discovery│ │ Rule      │ │ Rule Injector    │ ││ │
│  │  │  │ & Loading     │ │ Matching  │ │ (system prompt)  │ ││ │
│  │  │  └───────────────┘ └───────────┘ └──────────────────┘ ││ │
│  │  └────────────────────────────────────────────────────────┘│ │
│  │  ┌────────────────────────────────────────────────────────┐│ │
│  │  │         LLM Provider Abstraction                        ││ │
│  │  │  ┌────────┐ ┌────────┐ ┌────────┐ ┌─────────┐         ││ │
│  │  │  │OpenAI  │ │Vertex  │ │Bedrock │ │Self-    │         ││ │
│  │  │  │compat. │ │AI      │ │        │ │hosted   │         ││ │
│  │  │  └────────┘ └────────┘ └────────┘ └─────────┘         ││ │
│  │  │  Per-provider: ChatModel | EmbeddingModel              ││ │
│  │  └────────────────────────────────────────────────────────┘│ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  MCP Server (exposes KFP operations to external AI tools)  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │         Existing API Server (unchanged)                     │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

### LLM Integration Layer

#### Provider Abstraction

Inspired by Notebook Intelligence's model hierarchy, the provider abstraction separates **providers**
(the service endpoint) from **models** (the specific capability). A single provider can offer
multiple model types, and different tasks can use different models.

```go
// LLMProvider defines the interface for LLM backends.
// Inspired by NBI's LLMProvider which separates chat_models,
// inline_completion_models, and embedding_models under one provider.
type LLMProvider interface {
    // ChatModels returns the chat models available from this provider.
    ChatModels() []ChatModel
    // EmbeddingModels returns the embedding models available from this provider.
    EmbeddingModels() []EmbeddingModel
    // Name returns the provider name for logging and metrics.
    Name() string
    // DiscoverModels dynamically queries the provider for available models.
    // Inspired by NBI's Ollama provider which auto-discovers local models.
    DiscoverModels(ctx context.Context) error
}

// ChatModel represents a model capable of multi-turn conversation.
type ChatModel interface {
    // ChatCompletion sends a prompt and returns a response.
    ChatCompletion(ctx context.Context, req *ChatRequest) (*ChatResponse, error)
    // StreamChatCompletion sends a prompt and streams the response.
    StreamChatCompletion(ctx context.Context, req *ChatRequest) (<-chan *ChatChunk, error)
    // ID returns the model identifier (e.g., "gpt-4o", "claude-sonnet-4-5").
    ID() string
    // ContextWindow returns the model's maximum context window in tokens.
    // Used by the context builder to determine how much pipeline spec/log
    // content to include. Inspired by NBI's per-model context_window property.
    ContextWindow() int
    // SupportsTools returns whether this model supports native tool/function calling.
    // When false, the system falls back to prompt-based tool invocation.
    // Inspired by NBI's ChatModel.supports_tools property.
    SupportsTools() bool
}

// EmbeddingModel represents a model capable of generating vector embeddings.
// Used for RAG-based pipeline search and similar pipeline discovery.
type EmbeddingModel interface {
    // Embed generates embeddings for the given texts.
    Embed(ctx context.Context, texts []string) ([][]float32, error)
    // ID returns the model identifier.
    ID() string
    // Dimensions returns the embedding vector dimension.
    Dimensions() int
}
```

#### Model Capability Types

Different AI tasks can be routed to different models:

| Task | Model Type | Rationale |
|------|-----------|-----------|
| Chat / debugging / pipeline generation | `ChatModel` | Needs strong reasoning and code generation |
| Pipeline similarity search | `EmbeddingModel` | Vector search over pipeline descriptions |
| Inline suggestions (future) | `ChatModel` (lightweight) | Latency-sensitive, can use smaller model |

This separation allows organizations to use a powerful model (e.g., GPT-4o, Claude Sonnet) for
complex tasks while using a cheaper/faster model for simpler operations — a pattern NBI uses
extensively with its per-task model assignment.

#### Configuration

Configuration via `pipeline-install-config` ConfigMap:

```yaml
## Gen-AI configuration. Feature is disabled when aiProvider is empty (default).
aiProvider: ""              # "openai", "vertexai", "bedrock", "custom"
aiEndpoint: ""              # LLM API endpoint URL

## Model assignments — different models for different capabilities.
## Inspired by NBI's separate chat_model / embedding_model / inline_completion_model config.
aiChatModel: ""             # Model for chat/agent (e.g., "gpt-4o", "claude-sonnet-4-5")
aiEmbeddingModel: ""        # Model for embeddings (e.g., "text-embedding-3-small")
aiChatModelContextWindow: "128000"  # Context window for the chat model (tokens)

aiMaxTokensPerRequest: "4096"
aiRateLimitPerMinute: "30"

## Controls what pipeline context is sent to the LLM. Comma-separated.
## Options: "logs", "parameters", "metrics", "artifacts", "pipeline-spec"
aiAllowedContext: "logs,parameters,metrics,pipeline-spec"

## Default chat mode: "ask" (read-only) or "agent" (can take actions).
aiDefaultChatMode: "ask"

## Prompt rules: path to rules directory (ConfigMap mount or PVC).
aiRulesPath: ""             # e.g., "/etc/kfp-ai-rules"
aiRulesEnabled: "false"

## MCP: external MCP servers to connect to.
aiMCPConfigPath: ""         # Path to MCP server configuration JSON

## Extension tools: enable/disable the extension point for custom tools.
aiExtensionsEnabled: "false"
```

The API key or credentials are stored in a Kubernetes Secret, not the ConfigMap:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kfp-ai-credentials
type: Opaque
data:
  api-key: <base64-encoded-key>
```

### Agentic Tool System

The most significant architectural shift from the original proposal is replacing fixed RPCs
(`GeneratePipeline`, `AnalyzeRun`, `SuggestOptimizations`) with an **agentic tool system** where
the LLM dynamically decides which tools to invoke during a conversation.

This is directly inspired by Notebook Intelligence's tool architecture, which defines `Tool`,
`SimpleTool`, `Toolset`, and `SecuredExtensionTool` abstractions that chat participants invoke
during conversations.

#### Tool Abstraction

```go
// Tool represents an operation the AI agent can invoke during a conversation.
// Inspired by NBI's Tool/SimpleTool/Toolset hierarchy.
type Tool interface {
    // Name returns the tool's unique identifier.
    Name() string
    // Description returns a human-readable description for the LLM.
    Description() string
    // Parameters returns the JSON Schema for the tool's input parameters.
    Parameters() json.RawMessage
    // Execute runs the tool and returns the result.
    Execute(ctx context.Context, params json.RawMessage) (*ToolResult, error)
    // Mutating returns true if this tool modifies state (requires user confirmation).
    Mutating() bool
}

// ToolResult contains the output of a tool invocation.
type ToolResult struct {
    Content string `json:"content"`
    IsError bool   `json:"isError,omitempty"`
}

// Toolset groups related tools together, inspired by NBI's Toolset class.
type Toolset struct {
    Name  string
    Tools []Tool
}

// SecuredTool wraps a tool with user confirmation for mutating operations.
// Inspired by NBI's SecuredExtensionTool which wraps tools with a
// pre-invoke confirmation dialog.
type SecuredTool struct {
    Inner    Tool
    Approved bool // Set to true after user confirms
}
```

#### Built-in KFP Tools

The following tools are available to the AI agent out of the box:

| Tool | Category | Mutating | Description |
|------|----------|----------|-------------|
| `list_runs` | Read | No | List pipeline runs with filters (namespace, status, date range) |
| `get_run` | Read | No | Get detailed status of a specific run |
| `get_run_logs` | Read | No | Retrieve pod logs for a run's steps |
| `get_pipeline_spec` | Read | No | Get the compiled pipeline specification |
| `list_pipelines` | Read | No | List pipelines with filters |
| `get_pipeline` | Read | No | Get pipeline details and versions |
| `list_experiments` | Read | No | List experiments in a namespace |
| `get_resource_quotas` | Read | No | Check namespace resource quotas and usage |
| `compile_pipeline` | Read | No | Compile Python SDK code and report errors |
| `search_similar_pipelines` | Read | No | Find pipelines similar to a description (embedding-based) |
| `create_run` | Write | **Yes** | Submit a new pipeline run |
| `create_pipeline_version` | Write | **Yes** | Upload a new pipeline version |
| `stop_run` | Write | **Yes** | Terminate a running pipeline run |
| `delete_run` | Write | **Yes** | Delete a pipeline run |
| `create_experiment` | Write | **Yes** | Create a new experiment |

All read-only tools are **auto-approved** (the agent can invoke them freely). All mutating tools
are **wrapped with `SecuredTool`** and require explicit user confirmation in the UI before
execution — following NBI's pattern where `SecuredExtensionTool` shows a confirmation dialog
before `pre_invoke`.

```go
// Example: the agent decides to analyze a failed run.
// No fixed "AnalyzeRun" RPC — the agent composes tools dynamically:
//
// 1. Agent calls get_run(run_id) → auto-approved (read-only)
// 2. Agent sees step "train" failed, calls get_run_logs(run_id, step="train") → auto-approved
// 3. Agent calls get_pipeline_spec(pipeline_id) → auto-approved
// 4. Agent synthesizes root cause from logs + spec context
// 5. Agent suggests fix: "Increase memory to 8Gi"
// 6. If user says "do it", agent calls create_pipeline_version(...) → CONFIRMATION REQUIRED
//    → UI shows: "The AI wants to create a new pipeline version. Allow?"
//    → User clicks Approve / Deny
```

#### Tool Approval and Security

Inspired by NBI's `SecuredExtensionTool` pattern:

```go
// ToolApprovalPolicy defines the approval rules for tool invocations.
type ToolApprovalPolicy struct {
    // AutoApprove lists tool categories that don't require confirmation.
    // Default: ["read"]
    AutoApprove []string `json:"autoApprove"`
    // RequireConfirmation lists tool categories that always require confirmation.
    // Default: ["write", "delete", "external"]
    RequireConfirmation []string `json:"requireConfirmation"`
    // NeverAllow lists tools that are completely disabled.
    NeverAllow []string `json:"neverAllow"`
}
```

### Chat Modes

Inspired by NBI's distinct chat modes (`ask`, `agent`, `inline-chat`), the AI assistant supports
multiple interaction modes with different capabilities:

| Mode | Tool Access | Use Case | Behavior |
|------|------------|----------|----------|
| **Ask** | Read-only tools only | Q&A, explanations, analysis | Safe for exploration. The agent can read pipeline specs, logs, and metrics but cannot take any actions. Default mode. |
| **Agent** | All tools (with confirmation for writes) | Multi-step operations, pipeline creation, debugging with fix application | Full agent capabilities. The agent can compose read and write tools. Mutating tools require explicit user confirmation. |

The mode is selectable in the UI via a toggle in the AI assistant panel, defaulting to the
`aiDefaultChatMode` configuration value. This mirrors NBI's mode selector where users choose
between `ask` and `agent` modes with different tool access and behavior expectations.

```
┌─────────────────────────────────────────┐
│ AI Assistant           [Ask ▼] [Agent]  │
│─────────────────────────────────────────│
│                                         │
│  In Ask mode, I can read your pipelines │
│  and runs but won't make changes.       │
│  Switch to Agent mode to take actions.  │
│                                         │
└─────────────────────────────────────────┘
```

### Automatic Context Gathering

Inspired by NBI's `context_factory.py` which automatically assembles relevant context (current
file, directory, kernel type, cell contents) based on the user's current state, the AI service
automatically gathers KFP-specific context based on the page the user is viewing:

| UI Surface | Auto-Gathered Context |
|-----------|----------------------|
| **Run detail page** | Run status, step statuses, pod events, recent log lines (last 100 per failed step), pipeline spec, input parameters |
| **Pipeline detail page** | Pipeline spec, component list, recent run history (last 10), parameter definitions, default values |
| **Experiment page** | Experiment metadata, run list with status summary, aggregate metrics |
| **Pipeline list** | Namespace, available pipelines (names + descriptions), resource quotas |
| **Global (no specific page)** | User namespace, recent activity summary |

The context builder uses the chat model's `ContextWindow()` to determine how much context to
include, prioritizing the most relevant information and truncating gracefully:

```go
// ContextBuilder assembles page-specific context for LLM prompts.
// Inspired by NBI's context_factory.py / RuleContextFactory.
type ContextBuilder struct {
    runStore   storage.RunStoreInterface
    pipeStore  storage.PipelineStoreInterface
    logClient  LogClientInterface
}

// BuildContext gathers context based on the current UI page.
func (cb *ContextBuilder) BuildContext(ctx context.Context, req *ContextRequest) (*Context, error) {
    budget := req.Model.ContextWindow() - req.ReservedTokens

    var sections []ContextSection
    switch req.PageType {
    case PageRunDetail:
        sections = cb.buildRunDetailContext(ctx, req.RunID, budget)
    case PagePipelineDetail:
        sections = cb.buildPipelineDetailContext(ctx, req.PipelineID, budget)
    case PageExperiment:
        sections = cb.buildExperimentContext(ctx, req.ExperimentID, budget)
    default:
        sections = cb.buildGlobalContext(ctx, req.Namespace, budget)
    }

    return &Context{Sections: sections}, nil
}
```

### Prompt Rules System

Inspired by NBI's rules engine (`rule_manager.py`, `ruleset.py`, `rule_injector.py`), this
feature allows platform administrators to inject organization-specific guidelines into AI
prompts without modifying code or fine-tuning models.

#### Rule Definition Format

Rules are Markdown files with YAML frontmatter, following NBI's convention:

```markdown
---
# Rule metadata (YAML frontmatter)
apply: always          # "always", "auto" (matched by scope), or "manual"
active: true           # Can be toggled by admins
priority: 10           # Lower number = higher priority (evaluated first)
scope:
  namespaces:          # Apply only in these namespaces (empty = all)
    - "production"
    - "staging"
  chat_modes:          # Apply only in these modes (empty = all)
    - "agent"
  pipeline_patterns:   # Apply to pipelines matching these globs
    - "*-training-*"
---

# Production Safety Rules

When generating or modifying pipelines for production namespaces:

1. Always include resource limits (CPU and memory) on every container.
2. Use `kfp.kubernetes.use_secret_as_env()` for all credentials — never hardcode secrets.
3. Prefer our organization's base image `gcr.io/our-org/kfp-base:stable` over public images.
4. Include a notification step (Slack webhook) on pipeline failure.
5. Set `retry_policy` with max 3 retries and exponential backoff on all training steps.
```

#### Rule Storage and Discovery

Rules can be stored in:

1. **ConfigMap** (recommended for cluster-wide rules):
   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: kfp-ai-rules
   data:
     production-safety.md: |
       ---
       apply: always
       priority: 10
       scope:
         namespaces: ["production"]
       ---
       # Production Safety Rules
       ...
     cost-optimization.md: |
       ---
       apply: auto
       priority: 20
       ---
       # Cost Optimization Guidelines
       ...
   ```

2. **PVC mount** (for teams managing rules in version control):
   ```
   /etc/kfp-ai-rules/
   ├── global/
   │   ├── security.md
   │   └── best-practices.md
   └── modes/
       ├── ask/
       │   └── explanation-style.md
       └── agent/
           └── confirmation-rules.md
   ```

This mirrors NBI's rule discovery structure (`~/.jupyter/nbi/rules/` with `modes/` subdirectories).

#### Rule Injection

Rules are injected into the system prompt before each LLM call:

```go
// RuleInjector formats matched rules and appends them to the system prompt.
// Inspired by NBI's RuleInjector which formats rules with headers and
// priority-based ordering.
type RuleInjector struct {
    manager *RuleManager
}

func (ri *RuleInjector) InjectRules(systemPrompt string, ruleCtx *RuleContext) string {
    rules := ri.manager.MatchRules(ruleCtx)
    if len(rules) == 0 {
        return systemPrompt
    }

    var sb strings.Builder
    sb.WriteString(systemPrompt)
    sb.WriteString("\n\n## Organization Rules\n\n")
    sb.WriteString("The following rules are defined by the platform administrator. ")
    sb.WriteString("You MUST follow these rules when generating responses.\n\n")

    for _, rule := range rules {
        sb.WriteString(fmt.Sprintf("### %s (priority: %d)\n\n", rule.Name, rule.Priority))
        sb.WriteString(rule.Content)
        sb.WriteString("\n\n")
    }

    return sb.String()
}
```

### MCP Integration

The [Model Context Protocol (MCP)](https://modelcontextprotocol.io/) is an open standard for
connecting AI models to external tools and data sources. NBI has full MCP support with stdio and
HTTP transports. This KEP adds MCP integration to KFP in two directions.

#### KFP as MCP Server

KFP exposes its operations as an MCP server, allowing external AI tools (Claude Code, Cursor,
Windsurf, custom agents) to interact with KFP programmatically:

```json
{
  "name": "kubeflow-pipelines",
  "version": "1.0.0",
  "tools": [
    {
      "name": "list_runs",
      "description": "List pipeline runs in a namespace",
      "inputSchema": {
        "type": "object",
        "properties": {
          "namespace": {"type": "string"},
          "status": {"type": "string", "enum": ["succeeded", "failed", "running"]},
          "limit": {"type": "integer", "default": 10}
        }
      }
    },
    {
      "name": "get_run_logs",
      "description": "Get pod logs for a pipeline run's steps",
      "inputSchema": {
        "type": "object",
        "properties": {
          "run_id": {"type": "string"},
          "step_name": {"type": "string"},
          "tail_lines": {"type": "integer", "default": 100}
        },
        "required": ["run_id"]
      }
    },
    {
      "name": "submit_pipeline",
      "description": "Submit a compiled pipeline for execution",
      "inputSchema": {
        "type": "object",
        "properties": {
          "pipeline_spec": {"type": "string"},
          "namespace": {"type": "string"},
          "parameters": {"type": "object"}
        },
        "required": ["pipeline_spec"]
      }
    }
  ]
}
```

The MCP server is deployed as a sidecar or standalone service, exposing KFP operations over the
MCP Streamable HTTP transport. This makes KFP a first-class citizen in the MCP ecosystem — a
developer using Claude Code in their terminal can query KFP runs and submit pipelines without
leaving their editor.

#### Consuming External MCP Servers

The AI service can connect to external MCP servers, making their tools available to the AI agent
during conversations. Configuration follows NBI's pattern:

```json
{
  "mcpServers": {
    "model-registry": {
      "transport": "streamableHttp",
      "url": "http://model-registry.ml-platform:8080/mcp",
      "description": "Internal model registry for deployed models"
    },
    "data-catalog": {
      "transport": "streamableHttp",
      "url": "http://data-catalog.data-platform:8080/mcp",
      "description": "Data catalog for datasets and feature stores"
    },
    "cost-tracker": {
      "transport": "stdio",
      "command": "/usr/local/bin/cost-tracker-mcp",
      "args": ["--cluster", "production"],
      "description": "Cloud cost tracking for compute resources"
    }
  }
}
```

Tools from external MCP servers are **never auto-approved** — they are always wrapped with
`SecuredTool` and require user confirmation, following NBI's practice of wrapping MCP tools with
pre-invoke confirmation.

### Backend Changes

#### API Server Extensions

The AI service is restructured around the agentic architecture:

```go
// AIService handles Gen-AI operations with an agentic tool system.
type AIService struct {
    chatModel     ChatModel
    embedModel    EmbeddingModel
    toolRegistry  *ToolRegistry
    ruleManager   *RuleManager
    ruleInjector  *RuleInjector
    contextBuilder *ContextBuilder
    mcpManager    *MCPManager
    config        AIConfig
}

// ToolRegistry manages all available tools (built-in, MCP, extension).
type ToolRegistry struct {
    builtinTools   []Tool
    mcpTools       []Tool      // Discovered from MCP servers
    extensionTools []Tool      // Registered by extensions
    approvalPolicy ToolApprovalPolicy
}

// MCPManager handles connections to external MCP servers.
// Inspired by NBI's MCPManager with status tracking and tool discovery.
type MCPManager struct {
    servers map[string]*MCPServerConnection
    config  MCPConfig
}

type MCPServerConnection struct {
    Name      string
    Transport string // "streamableHttp" or "stdio"
    Status    MCPStatus
    Tools     []Tool
}

type MCPStatus string

const (
    MCPNotConnected    MCPStatus = "NotConnected"
    MCPConnecting      MCPStatus = "Connecting"
    MCPConnected       MCPStatus = "Connected"
    MCPFailedToConnect MCPStatus = "FailedToConnect"
)
```

#### Proto Definitions

The proto definitions are updated to support the agentic architecture with rich streaming
responses:

```protobuf
service AIService {
  // Primary agentic chat endpoint. The LLM dynamically invokes tools
  // during the conversation. Replaces the fixed-RPC approach.
  rpc Chat(ChatRequest) returns (stream ChatStreamEvent);

  // Convenience RPCs that internally use the agentic chat with pre-configured
  // prompts. These exist for programmatic access (SDK, CI/CD) where the full
  // chat UX is not needed.
  rpc GeneratePipeline(GeneratePipelineRequest) returns (GeneratePipelineResponse);
  rpc GenerateDocumentation(GenerateDocumentationRequest) returns (GenerateDocumentationResponse);

  // Tool approval: user responds to a confirmation request.
  rpc ApproveToolCall(ToolApprovalRequest) returns (ToolApprovalResponse);

  // MCP server management.
  rpc ListMCPServers(ListMCPServersRequest) returns (ListMCPServersResponse);

  // Rule management.
  rpc ListRules(ListRulesRequest) returns (ListRulesResponse);
  rpc ToggleRule(ToggleRuleRequest) returns (ToggleRuleResponse);
}

// --- Chat ---

message ChatRequest {
  string message = 1;
  string conversation_id = 2;  // For multi-turn context
  string namespace = 3;        // Scope queries to a namespace
  ChatMode mode = 4;           // Ask or Agent mode
  PageContext page_context = 5; // Current UI page for auto context gathering
  RequestToolSelection tool_selection = 6; // Which tools to enable
}

enum ChatMode {
  CHAT_MODE_ASK = 0;     // Read-only tools, safe exploration
  CHAT_MODE_AGENT = 1;   // All tools, confirmation for writes
}

message PageContext {
  PageType page_type = 1;
  string run_id = 2;
  string pipeline_id = 3;
  string pipeline_version_id = 4;
  string experiment_id = 5;
}

enum PageType {
  PAGE_GLOBAL = 0;
  PAGE_RUN_DETAIL = 1;
  PAGE_PIPELINE_DETAIL = 2;
  PAGE_EXPERIMENT = 3;
  PAGE_PIPELINE_LIST = 4;
}

message RequestToolSelection {
  bool builtin_tools = 1;    // KFP built-in tools (default: true)
  bool mcp_tools = 2;        // External MCP server tools (default: false)
  bool extension_tools = 3;  // Custom extension tools (default: false)
}

// Rich streaming response inspired by NBI's ResponseStreamData which supports
// Markdown, Image, HTMLFrame, Button, Anchor, Progress, Confirmation, etc.
message ChatStreamEvent {
  oneof event {
    MarkdownChunk markdown = 1;           // Streaming text content
    ToolCallEvent tool_call = 2;          // Agent is invoking a tool
    ToolResultEvent tool_result = 3;      // Tool returned a result
    ConfirmationRequest confirmation = 4; // Agent needs user approval
    ProgressEvent progress = 5;           // Long-running operation progress
    ActionButton action_button = 6;       // Clickable action in the response
    AnchorLink anchor_link = 7;           // Link to a KFP resource
    CodeBlock code_block = 8;             // Syntax-highlighted code with copy button
    FinalEvent final = 9;                 // Conversation turn complete
  }
}

message MarkdownChunk {
  string content = 1;
}

message ToolCallEvent {
  string tool_call_id = 1;
  string tool_name = 2;
  string parameters_json = 3;
  bool requires_approval = 4;  // True for mutating tools
}

message ToolResultEvent {
  string tool_call_id = 1;
  string result = 2;
  bool is_error = 3;
}

message ConfirmationRequest {
  string confirmation_id = 1;
  string tool_name = 2;
  string description = 3;      // Human-readable description of the action
  string parameters_json = 4;  // What will be executed
}

message ProgressEvent {
  string message = 1;
  float percentage = 2;  // 0.0 to 1.0, -1 for indeterminate
}

message ActionButton {
  string label = 1;
  string action = 2;     // Frontend action identifier
  string payload = 3;    // JSON payload for the action
}

message AnchorLink {
  string label = 1;
  string resource_type = 2; // "run", "pipeline", "experiment"
  string resource_id = 3;
  string url = 4;
}

message CodeBlock {
  string language = 1;   // "python", "yaml", etc.
  string code = 2;
  bool compilable = 3;   // For pipeline code: did it pass compilation?
}

message FinalEvent {
  int32 tokens_used = 1;
  string conversation_id = 2;
}

// --- Tool Approval ---

message ToolApprovalRequest {
  string confirmation_id = 1;
  string conversation_id = 2;
  bool approved = 3;
}

message ToolApprovalResponse {
  bool success = 1;
}

// --- Convenience RPCs ---

message GeneratePipelineRequest {
  string description = 1;
  string sdk_version = 2;
  repeated string components = 3;
}

message GeneratePipelineResponse {
  string python_code = 1;
  string explanation = 2;
  repeated string warnings = 3;
  bool compiles = 4;
}

message GenerateDocumentationRequest {
  string pipeline_id = 1;
  string pipeline_version_id = 2;
  string format = 3;
}

message GenerateDocumentationResponse {
  string documentation = 1;
}

// --- MCP ---

message ListMCPServersRequest {}

message ListMCPServersResponse {
  repeated MCPServerInfo servers = 1;
}

message MCPServerInfo {
  string name = 1;
  string status = 2;      // "Connected", "Connecting", "FailedToConnect"
  string description = 3;
  repeated string tool_names = 4;
}

// --- Rules ---

message ListRulesRequest {}

message ListRulesResponse {
  repeated RuleInfo rules = 1;
}

message RuleInfo {
  string name = 1;
  string file_path = 2;
  int32 priority = 3;
  bool active = 4;
  string apply_mode = 5;  // "always", "auto", "manual"
  repeated string scoped_namespaces = 6;
}

message ToggleRuleRequest {
  string name = 1;
  bool active = 2;
}

message ToggleRuleResponse {
  bool success = 1;
}
```

### Frontend Changes

#### AI Assistant Panel

A collapsible side panel accessible from a toolbar button throughout the UI. Updated to support
the agentic architecture:

- A chat interface with **mode selector** (Ask / Agent) — inspired by NBI's mode toggle.
- **Rich streaming display** supporting all `ChatStreamEvent` types (see below).
- Context-aware prompts based on the current page (e.g., on a failed run page, the panel
  pre-populates "Analyze this run failure").
- **Tool call visibility**: When the agent invokes tools, each call is shown in the chat
  (tool name, parameters, result) so the user can follow the agent's reasoning.
- **Confirmation dialogs**: When the agent requests approval for a mutating tool, an inline
  confirmation widget appears with Approve / Deny buttons.

The panel is only rendered when the AI feature is enabled (detected via a feature flag endpoint).

#### Rich Streaming Responses

Inspired by NBI's `ResponseStreamData` which supports Markdown, Image, HTMLFrame, Button, Anchor,
Progress, Confirmation, and AskUserQuestion content types, the chat stream supports rich
interactive elements:

| Stream Event | UI Rendering |
|-------------|-------------|
| `MarkdownChunk` | Streaming text with Markdown rendering |
| `ToolCallEvent` | Collapsible "Tool: list_runs(...)" block showing parameters |
| `ToolResultEvent` | Collapsible result block under the tool call |
| `ConfirmationRequest` | Inline card with action description + Approve / Deny buttons |
| `ProgressEvent` | Progress bar or spinner with message |
| `ActionButton` | Clickable button that triggers a frontend action (e.g., "Open Run") |
| `AnchorLink` | Inline link to a KFP resource (run, pipeline, experiment) |
| `CodeBlock` | Syntax-highlighted code with Copy button and compilation badge |

Example rendering of an agent conversation:

```
┌────────────────────────────────────────────────────┐
│ AI Assistant                     [Ask] [Agent ●]   │
│────────────────────────────────────────────────────│
│                                                    │
│ You: Why did run abc-123 fail?                     │
│                                                    │
│ Agent:                                             │
│ ▶ Tool: get_run("abc-123")                    ✓    │
│ ▶ Tool: get_run_logs("abc-123", step="train") ✓    │
│                                                    │
│ The run failed due to an OOM error in the          │
│ **train** step. The container requested 4Gi but    │
│ the training dataset required ~7Gi in memory.      │
│                                                    │
│ **Suggested fix**: Increase memory request to 8Gi. │
│                                                    │
│ ```python                                          │
│ @dsl.container_component                           │
│ def train(...):                                    │
│     return dsl.ContainerSpec(                      │
│         ...                                        │
│         resources=dsl.Resources(memory="8Gi")      │
│     )                                   [Copy]     │
│ ```                                     ✓ Compiles │
│                                                    │
│ [Open Run abc-123]  [View Pipeline]                │
│                                                    │
│ You: Apply this fix and submit a new run           │
│                                                    │
│ Agent:                                             │
│ ▶ Tool: create_pipeline_version(...)               │
│ ┌────────────────────────────────────────────┐     │
│ │ ⚠ The AI wants to create a new pipeline   │     │
│ │ version "v2-memory-fix" in namespace       │     │
│ │ "ml-team".                                 │     │
│ │                                            │     │
│ │         [Approve]    [Deny]                │     │
│ └────────────────────────────────────────────┘     │
│                                                    │
└────────────────────────────────────────────────────┘
```

**Interactive UI Mockup:**

The following screenshot shows a working mockup of the AI Assistant panel analyzing a failed pipeline run, demonstrating tool call visualization, root cause analysis, code suggestions with compilation badges, and action buttons:

![KFP Gen AI Assistant Mockup](images/kfp-genai-assistant-mockup.png)

#### Inline AI Features

Context-specific AI actions integrated into existing UI surfaces:

- **Run details page**: "Analyze failure" button on failed runs (opens chat in Ask mode with
  pre-filled prompt and auto-gathered run context).
- **Pipeline details page**: "Generate docs" and "Suggest optimizations" buttons.
- **Pipeline list**: "Create with AI" button alongside the existing upload flow.

### Extension Point Architecture

Inspired by NBI's `NotebookIntelligenceExtension` interface which allows third-party extensions to
register custom toolsets, the AI service exposes an extension point for platform teams to register
custom tools.

This addresses the "External Plugin Architecture" alternative from the original KEP — rather than
rejecting extensibility entirely, we build the core features in-house **and** expose a controlled
extension point.

```go
// AIExtension allows platform teams to register custom tools
// that the AI agent can invoke during conversations.
// Inspired by NBI's NotebookIntelligenceExtension interface.
type AIExtension interface {
    // Name returns the extension identifier.
    Name() string
    // Tools returns the tools this extension provides.
    Tools() []Tool
}

// Registration happens via a Go plugin or a sidecar gRPC service.
// Example: a team registers their model registry as an extension:
//
//   type ModelRegistryExtension struct{}
//
//   func (e *ModelRegistryExtension) Name() string { return "model-registry" }
//   func (e *ModelRegistryExtension) Tools() []Tool {
//       return []Tool{
//           &SimpleTool{
//               name: "lookup_model",
//               desc: "Look up a model by name in the model registry",
//               mutating: false,
//               handler: func(ctx context.Context, params json.RawMessage) (*ToolResult, error) {
//                   // Query model registry...
//               },
//           },
//       }
//   }
```

Extension tools are always wrapped with `SecuredTool` (require user confirmation) unless
explicitly marked as safe by the administrator.

### SDK Extensions

An optional `kfp[ai]` extra dependency provides local AI-assisted authoring:

```python
from kfp.ai import generate_pipeline

# Generate a pipeline from a description
code = generate_pipeline(
    description="Load CSV from GCS, train a scikit-learn random forest, evaluate, and deploy",
    sdk_version="2.15.0",
)
print(code)
```

This is a convenience wrapper that calls the API server's `GeneratePipeline` endpoint. It is not
required — the UI provides the same functionality.

For local-only use (no KFP backend), the SDK can call the LLM provider directly using the user's
own API key:

```python
from kfp.ai import generate_pipeline

code = generate_pipeline(
    description="...",
    provider="openai",  # Uses OPENAI_API_KEY env var
    model="gpt-4o",
)
```

### Security and Privacy

- **Opt-in only**: The entire feature is disabled by default. No LLM calls are made unless an
  administrator explicitly configures the provider.
- **Credential isolation**: API keys are stored in Kubernetes Secrets, never in ConfigMaps or
  environment variables logged to stdout.
- **Context filtering**: Administrators configure which categories of pipeline data
  (logs, parameters, metrics, artifacts, pipeline specs) may be sent to the LLM provider.
- **Secret redaction**: Environment variable values, secret references, and credential-like strings
  are automatically redacted before inclusion in LLM prompts.
- **Prompt injection defense**: All user-provided content (pipeline names, parameters, log lines)
  is sanitized and placed in structured prompt sections with clear system/user boundaries.
  Prompt rules from admins are injected in a dedicated system prompt section, separated from
  user content.
- **Audit logging**: All AI service calls are logged with the requesting user, namespace, operation
  type, and token count (but not the prompt or response content). Tool invocations are logged
  individually, including whether they were auto-approved or user-confirmed.
- **RBAC**: AI endpoints respect existing KFP RBAC. In multi-user mode, users can only query
  pipelines and runs within their authorized namespaces.
- **Secured tool wrapping**: All mutating tools and all external tools (MCP, extensions) require
  explicit user confirmation before execution. This is enforced at the framework level, not per
  tool, preventing a tool author from accidentally bypassing the confirmation requirement.
- **MCP server allowlisting**: Only MCP servers explicitly configured by the administrator are
  connected. There is no auto-discovery of MCP servers on the network.

### Test Plan

#### Prerequisite testing updates

- [ ] I/we understand the owners of the involved components may require updates to existing tests
  to make this code solid enough prior to review.

#### Unit Tests

| Package | Coverage Target | Key Tests |
|---------|----------------|-----------|
| `backend/src/apiserver/ai` | 80%+ | Provider abstraction, model capability routing, context building, secret redaction, rate limiting |
| `backend/src/apiserver/ai/providers` | 80%+ | Each provider implementation (mock HTTP responses), error handling, retry logic, model discovery |
| `backend/src/apiserver/ai/tools` | 80%+ | Tool registry, tool execution, SecuredTool wrapping, approval policy enforcement |
| `backend/src/apiserver/ai/rules` | 80%+ | Rule parsing (Markdown + YAML frontmatter), scope matching, priority ordering, rule injection into prompts |
| `backend/src/apiserver/ai/mcp` | 80%+ | MCP server connection, tool discovery, transport handling (HTTP + stdio), status management |
| `frontend/src/components/AIAssistant` | 70%+ | Panel rendering, mode switching, streaming display, confirmation dialogs, tool call visualization, rich response types |

#### Integration Tests

- API server with a mock LLM provider verifying end-to-end agentic chat flow (tool calls,
  confirmations, streaming responses).
- Tool execution: verify read-only tools are auto-approved, mutating tools block until confirmed.
- Context builder: verify correct context is gathered for each page type.
- Rule injection: verify rules are loaded, matched, and injected into prompts correctly.
- MCP integration: verify tool discovery from a mock MCP server and tool execution flow.
- Extension tools: verify custom tools are registered and wrapped with SecuredTool.
- RBAC enforcement on AI endpoints in multi-user mode.
- Rate limiting behavior under concurrent requests.
- GeneratePipeline → compile validation → response flow.
- AnalyzeRun via agentic chat with seeded failed run data.

#### E2E Tests

- Enable AI feature via ConfigMap, verify UI panel appears with mode selector.
- Disable AI feature, verify UI panel is hidden and endpoints return 404.
- Chat in Ask mode: verify only read-only tools are available.
- Chat in Agent mode: verify mutating tools show confirmation dialogs.
- User approves tool call → action executes successfully.
- User denies tool call → agent acknowledges and suggests alternatives.
- Generate a pipeline from natural language, compile it, and verify it produces a valid workflow.
- Analyze a deliberately failed run via agentic chat and verify the agent calls get_run + get_run_logs.
- Rich stream rendering: verify all event types (markdown, tool calls, confirmations, progress,
  action buttons, anchor links, code blocks) render correctly in the UI.
- Rule toggle: toggle a rule inactive, verify it is no longer injected into prompts.
- MCP server status display in admin panel.

### Graduation Criteria

#### Alpha

- Feature is behind a feature flag (ConfigMap `aiProvider` must be explicitly set).
- OpenAI-compatible provider implemented and tested with ChatModel and EmbeddingModel separation.
- Agentic chat endpoint with built-in KFP tools (read-only subset: `list_runs`, `get_run`,
  `get_run_logs`, `get_pipeline_spec`, `list_pipelines`).
- Ask mode only (no mutating tools).
- UI AI assistant panel with chat interface and tool call visualization.
- Basic auto context gathering for run detail and pipeline detail pages.
- Rich streaming responses (Markdown, CodeBlock, AnchorLink).
- Basic secret redaction in prompts.
- Documentation for administrator setup.

#### Beta

- All four provider implementations (OpenAI, Vertex AI, Bedrock, self-hosted) with model
  discovery support.
- Agent mode with mutating tools (`create_run`, `create_pipeline_version`, etc.) and
  confirmation dialogs.
- Prompt rules system with ConfigMap-based rule storage, scope matching, and admin toggle.
- MCP server support (KFP as MCP server + consuming external MCP servers).
- Extension point for custom tools.
- Full rich streaming responses (all event types including ConfirmationRequest, ProgressEvent,
  ActionButton).
- Auto context gathering for all page types.
- Rate limiting and token budget enforcement.
- Audit logging for all AI operations and tool invocations.
- Comprehensive context filtering controls.
- SDK `kfp[ai]` extension.
- Inline AI features on run/pipeline detail pages.

#### Stable

- Proven in production deployments with feedback incorporated.
- Performance benchmarks for prompt latency and token usage.
- Hardened prompt injection defenses with adversarial testing.
- Rule conflict detection and validation tooling.
- MCP server health monitoring and auto-reconnection.
- Localization support for non-English interactions.
- Full E2E test coverage in CI.

## Frontend Considerations

This proposal has significant frontend impact:

- **New AI assistant panel**: A new collapsible side panel component accessible from the main
  toolbar. This is the largest frontend addition.
- **Mode selector**: Toggle between Ask and Agent modes, with visual indicator of current mode.
- **Rich streaming renderer**: A component that handles all `ChatStreamEvent` types — markdown
  rendering, tool call visualization, confirmation dialogs, progress bars, action buttons,
  anchor links, and code blocks with syntax highlighting. This is more complex than a simple
  text stream and is inspired by NBI's `ResponseStreamData` renderer.
- **Confirmation dialog**: Inline card within the chat stream that shows the proposed action
  and Approve / Deny buttons. Must handle the async approval flow (chat pauses while waiting
  for user input).
- **Inline action buttons**: New buttons on run details, pipeline details, and pipeline list pages.
- **Feature flag gating**: All AI UI elements must be conditionally rendered based on a feature
  flag endpoint (`/apis/v2beta1/ai/enabled`). When the feature is disabled, there must be zero
  visual or functional changes to the existing UI.
- **Streaming response handling**: The chat interface must handle gRPC server-side streaming for
  real-time response display.
- **Code rendering**: Generated pipeline code must be displayed with Python syntax highlighting,
  a copy-to-clipboard action, and a compilation status badge.
- **Loading states**: LLM responses may take 5-30 seconds. The UI must provide appropriate
  progress indicators and allow cancellation.
- **Error handling**: Graceful handling of LLM provider errors, rate limit exceeded, and timeout
  scenarios.

No changes to existing frontend components are required. The AI features are purely additive.

## KFP Local Considerations

The Gen-AI features are backend-dependent and require the API server with LLM provider
configuration. They are **not available in KFP local mode** (local compilation and execution
without a remote backend).

However, the SDK `kfp[ai]` extension (Beta milestone) could support a local mode where the user
provides their own LLM API key via environment variable, enabling local pipeline generation without
a deployed KFP backend. This local SDK feature would call the LLM provider directly, bypassing the
API server.

## Migration Strategy

No migration is required. This is a purely additive feature with no changes to existing APIs,
data models, or workflows. Existing deployments are unaffected when the feature is not enabled.

When upgrading to a version that includes this feature:

- The new ConfigMap keys default to empty (feature disabled).
- No new environment variables are required unless the feature is enabled.
- No database schema changes are required.
- No changes to existing pipeline specs or runs.

## Implementation History

- 2026-02-19: Initial proposal.
- 2026-02-21: Major revision incorporating inspirations from Notebook Intelligence — agentic tool
  system, prompt rules engine, MCP integration, chat modes, rich streaming responses, extension
  point architecture, and enhanced provider abstraction.

## Drawbacks

- **Increased API server complexity**: Adding an LLM integration layer with tools, rules, and
  MCP increases the surface area of the API server. This is mitigated by the modular design —
  the AI service is a separate package that is only initialized when enabled.
- **External dependency on LLM providers**: The feature depends on external LLM APIs which may
  have availability, latency, or cost issues. This is mitigated by supporting self-hosted models
  and ensuring graceful degradation.
- **Maintenance burden for prompt templates and rules**: As the KFP SDK evolves, prompt templates
  and built-in rules that reference SDK APIs must be updated. This requires ongoing maintenance.
- **User trust in generated code**: Users may over-trust AI-generated pipeline code without
  adequate review. The UI should include clear disclaimers and encourage review.
- **Tool system complexity**: The agentic tool system is more complex to implement and test than
  fixed RPCs. However, it provides significantly more flexibility and reduces the need for new
  backend endpoints as capabilities are added.
- **MCP ecosystem maturity**: MCP is still an evolving standard. Breaking changes in the protocol
  may require updates. Mitigated by depending on stable MCP SDK versions.
- **Potential for scope creep**: Gen-AI features can expand indefinitely. The graduation criteria
  define a bounded scope for each phase.

## Alternatives

### Fixed RPC-Per-Use-Case (No Agent Tools)

The original design proposed dedicated RPCs for each use case (`GeneratePipeline`, `AnalyzeRun`,
`SuggestOptimizations`, `GenerateDocumentation`, `Chat`).

**Pros**: Simpler to implement; each endpoint is self-contained; easier to test in isolation.
**Cons**: Rigid — every new capability requires a new backend endpoint. The LLM cannot compose
operations dynamically (e.g., "check the logs, then check resource quotas, then suggest a fix").
Cannot leverage tool calling, which is a core strength of modern LLMs.

**Decision**: Rejected as the primary architecture. Convenience RPCs are retained for programmatic
access (`GeneratePipeline`, `GenerateDocumentation`) but the primary user-facing experience is
the agentic chat endpoint. This approach is validated by NBI's architecture, where the
`BaseChatParticipant` and `ClaudeCodeChatParticipant` use tool calling as the core interaction
pattern rather than fixed operations.

### SDK-Only Approach

Provide AI capabilities exclusively through the Python SDK (e.g., `kfp.ai.generate_pipeline()`)
without any UI or backend changes.

**Pros**: Simpler implementation; no frontend or API server changes.
**Cons**: Excludes UI-only users; cannot leverage server-side context (run logs, metrics, pipeline
history) for analysis and optimization; each user must configure their own LLM credentials.

**Decision**: Rejected as the sole approach. The SDK extension is included as a complementary
feature (Beta milestone), but the primary experience is through the UI and backend where
server-side context is available.

### Fine-Tuned Domain Model

Train or fine-tune a model specifically on KFP documentation, SDK code, and pipeline examples
rather than using general-purpose LLMs with prompt engineering.

**Pros**: Potentially higher accuracy for KFP-specific tasks; no dependency on external providers.
**Cons**: Significant training infrastructure and data curation effort; ongoing retraining as the
SDK evolves; large model hosting requirements; limited by training data rather than adaptable via
prompts.

**Decision**: Rejected for the initial implementation. RAG with general-purpose models, combined
with the prompt rules system for organization-specific knowledge, provides a better cost/quality
tradeoff. A fine-tuned model could be explored as a future enhancement if prompt engineering
proves insufficient.
