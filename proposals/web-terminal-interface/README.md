# KEP: Web-Based Terminal Interface for Kubeflow Pipelines

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Architecture Overview](#architecture-overview)
- [Design Details](#design-details)
  - [Backend: Terminal Server](#backend-terminal-server)
  - [Frontend: Web Terminal UI](#frontend-web-terminal-ui)
  - [Authentication and Authorization](#authentication-and-authorization)
  - [Session Management](#session-management)
  - [Security Model](#security-model)
- [Test Plan](#test-plan)
- [Migration Strategy](#migration-strategy)
- [Frontend Considerations](#frontend-considerations)
- [KFP Local Considerations](#kfp-local-considerations)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Summary

This proposal introduces a web-based, VS Code-like terminal interface embedded in the Kubeflow Pipelines UI. It allows pipeline users to execute commands directly against the Kubernetes cluster from within the KFP dashboard — eliminating the need to switch between the browser and a local terminal, install CLI tools, or manage kubeconfig files. The interface provides an integrated development and debugging experience for pipeline authors, including file browsing, a multi-tab terminal emulator, and an optional lightweight code editor pane.

## Motivation

Today, Kubeflow Pipelines users frequently need to:

1. **Debug failed pipeline runs** by inspecting pod logs, describing pods, and checking events — all of which require `kubectl` access outside the KFP UI.
2. **Iterate on pipeline code** by editing Python scripts, recompiling pipelines, and submitting them — a workflow that requires switching between an IDE, a terminal, and the KFP dashboard.
3. **Inspect cluster state** to understand resource availability (GPUs, PVCs, secrets) that affects pipeline execution.
4. **Manage artifacts and data** stored in PVCs or object storage that pipelines produce or consume.

Each of these tasks forces users out of the KFP UI and into a separately configured terminal environment. This context-switching is costly, especially for data scientists and ML engineers who may not have deep Kubernetes expertise. A web-based terminal embedded directly in the dashboard would collapse these workflows into a single pane of glass.

### Goals

1. Provide a web-based terminal accessible from the KFP dashboard that can execute commands against the Kubernetes cluster.
2. Support multi-tab terminal sessions with session persistence across page navigations.
3. Integrate a lightweight file browser and optional code editor (Monaco-based) for viewing and editing pipeline source files.
4. Respect existing Kubeflow RBAC and Kubernetes RBAC — users should only be able to perform actions they are authorized for.
5. Provide contextual entry points: e.g., open a terminal pre-configured in the namespace of a specific pipeline run with relevant environment variables set.
6. Support copy/paste, terminal resizing, color output, and standard terminal UX expectations.

### Non-Goals

1. Replacing full-featured IDEs or notebook environments (e.g., Jupyter, VS Code Server). This is a lightweight, focused terminal experience.
2. Providing SSH access to cluster nodes.
3. Supporting arbitrary long-running workloads in the terminal (sessions are ephemeral by design).
4. Multi-cluster terminal access (scoped to the cluster hosting the KFP deployment).
5. Providing a built-in pipeline SDK or compiler — users bring their own tools via the terminal.

## Proposal

### User Stories

#### Story 1: Debugging a Failed Run
As a **pipeline author**, I want to click a "Terminal" button on a failed run's detail page so that a terminal opens pre-configured in the run's namespace. I can immediately run `kubectl describe pod <pod-name>` and `kubectl logs <pod-name>` without leaving the dashboard or configuring kubeconfig locally.

#### Story 2: Quick Pipeline Submission
As a **data scientist**, I want to open the integrated editor from the KFP dashboard, modify a pipeline Python file stored in a shared PVC, and run `kfp pipeline create` directly in the terminal — all without leaving the browser.

#### Story 3: Cluster Resource Inspection
As a **platform engineer**, I want to check GPU availability (`kubectl get nodes -l gpu=true`) and PVC status (`kubectl get pvc`) from the KFP UI before configuring a pipeline that depends on these resources.

#### Story 4: Artifact Inspection
As a **ML engineer**, I want to browse output artifacts from a completed run by navigating to the relevant PVC mount in the file browser and previewing files — without needing to set up port-forwarding or object storage CLI tools locally.

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                   KFP Frontend (React)                  │
│  ┌──────────────┐  ┌────────────┐  ┌────────────────┐  │
│  │  Terminal UI  │  │   Editor   │  │  File Browser  │  │
│  │   (xterm.js)  │  │  (Monaco)  │  │   (Tree View)  │  │
│  └──────┬───────┘  └─────┬──────┘  └───────┬────────┘  │
│         │                │                  │           │
│         └────────────────┴──────────────────┘           │
│                          │ WebSocket / REST              │
└──────────────────────────┼──────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────┐
│              Terminal Server (Go)                        │
│  ┌─────────────┐  ┌─────────────┐  ┌────────────────┐  │
│  │  WebSocket   │  │   Session   │  │    RBAC        │  │
│  │  Handler     │  │   Manager   │  │    Enforcer    │  │
│  └──────┬──────┘  └──────┬──────┘  └───────┬────────┘  │
│         │                │                  │           │
│         └────────────────┴──────────────────┘           │
│                          │                              │
│              Kubernetes API (exec into Pod)              │
└──────────────────────────┼──────────────────────────────┘
                           │
┌──────────────────────────┼──────────────────────────────┐
│           Kubernetes Cluster                            │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Terminal Pod (per-user, ephemeral)               │   │
│  │  - Runs as user's ServiceAccount                  │   │
│  │  - Pre-installed tools: kubectl, kfp CLI, python  │   │
│  │  - Resource limits enforced                       │   │
│  │  - Auto-cleanup after idle timeout                │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

The system consists of three layers:

1. **Frontend**: xterm.js-based terminal emulator, Monaco editor, and file tree — embedded as a collapsible panel in the KFP dashboard.
2. **Terminal Server**: A new Go microservice that manages WebSocket connections, maps them to Kubernetes pod exec sessions, and enforces authorization.
3. **Terminal Pods**: Ephemeral, per-user pods with pre-installed tooling. Each pod runs under the user's ServiceAccount and is garbage-collected after an idle timeout.

## Design Details

### Backend: Terminal Server

A new Go service (`terminal-server`) will be deployed alongside the existing KFP backend services. It handles:

#### API Endpoints

```
POST   /api/v1/terminal/sessions          # Create a new terminal session
GET    /api/v1/terminal/sessions           # List active sessions
DELETE /api/v1/terminal/sessions/{id}      # Terminate a session
GET    /api/v1/terminal/sessions/{id}/ws   # WebSocket attach to session
POST   /api/v1/terminal/sessions/{id}/resize  # Resize terminal
GET    /api/v1/terminal/files?path=...     # List files in terminal pod
GET    /api/v1/terminal/files/content?path=...  # Read file content
PUT    /api/v1/terminal/files/content      # Write file content
```

#### Terminal Pod Spec

When a user creates a session, the terminal server spawns (or reuses) an ephemeral pod:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: terminal-{user-id}-{session-hash}
  namespace: {user-namespace}
  labels:
    app: kfp-terminal
    user: {user-id}
  annotations:
    kfp-terminal/idle-timeout: "30m"
    kfp-terminal/max-lifetime: "8h"
spec:
  serviceAccountName: {user-service-account}
  automountServiceAccountToken: true
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
  containers:
  - name: terminal
    image: gcr.io/kubeflow-images/kfp-terminal:latest
    command: ["/bin/sleep", "infinity"]
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 512Mi
    volumeMounts:
    - name: workspace
      mountPath: /home/user/workspace
  volumes:
  - name: workspace
    emptyDir:
      sizeLimit: 1Gi
  terminationGracePeriodSeconds: 5
```

The terminal container image includes:
- `kubectl` (matching cluster version)
- `kfp` CLI (matching KFP deployment version)
- `python3`, `pip`
- `bash`, `vim`, `less`, `curl`, `jq`
- `git`

#### Session Lifecycle

```
User clicks "Terminal"
        │
        ▼
Terminal Server receives POST /sessions
        │
        ├── Check user auth (via Kubeflow auth headers)
        ├── Check if existing idle pod exists for user
        │     ├── Yes → Reuse pod
        │     └── No  → Create new terminal pod
        │
        ▼
Pod reaches Running state
        │
        ▼
User connects via WebSocket /sessions/{id}/ws
        │
        ▼
Terminal Server calls Kubernetes exec API:
  POST /api/v1/namespaces/{ns}/pods/{pod}/exec
    ?command=/bin/bash
    &stdin=true&stdout=true&stderr=true&tty=true
        │
        ▼
Bidirectional stream: WebSocket ↔ Pod exec
        │
        ▼
On disconnect or idle timeout → Pod garbage collected
```

#### Go Types

```go
// Session represents an active terminal session.
type Session struct {
    ID          string            `json:"id"`
    UserID      string            `json:"userId"`
    Namespace   string            `json:"namespace"`
    PodName     string            `json:"podName"`
    Status      SessionStatus     `json:"status"`
    CreatedAt   time.Time         `json:"createdAt"`
    LastActive  time.Time         `json:"lastActive"`
    Context     map[string]string `json:"context,omitempty"`
}

type SessionStatus string

const (
    SessionPending    SessionStatus = "Pending"
    SessionRunning    SessionStatus = "Running"
    SessionTerminated SessionStatus = "Terminated"
)

// CreateSessionRequest is the request body for POST /sessions.
type CreateSessionRequest struct {
    Namespace   string            `json:"namespace"`
    Context     map[string]string `json:"context,omitempty"`
    // Context can include run_id, pipeline_id, etc.
    // to pre-configure the terminal environment.
}

// TerminalServerConfig holds configuration for the terminal server.
type TerminalServerConfig struct {
    MaxSessionsPerUser  int           `json:"maxSessionsPerUser"`
    IdleTimeout         time.Duration `json:"idleTimeout"`
    MaxSessionLifetime  time.Duration `json:"maxSessionLifetime"`
    TerminalImage       string        `json:"terminalImage"`
    DefaultCPURequest   string        `json:"defaultCpuRequest"`
    DefaultMemoryRequest string       `json:"defaultMemoryRequest"`
    DefaultCPULimit     string        `json:"defaultCpuLimit"`
    DefaultMemoryLimit  string        `json:"defaultMemoryLimit"`
}
```

### Frontend: Web Terminal UI

The terminal UI is embedded as a resizable bottom panel in the KFP dashboard (similar to VS Code's integrated terminal).

#### Key Components

| Component | Library | Purpose |
|-----------|---------|---------|
| Terminal Emulator | [xterm.js](https://xtermjs.org/) | Full terminal emulation with WebGL renderer |
| Code Editor | [Monaco Editor](https://microsoft.github.io/monaco-editor/) | Syntax-highlighted editing for Python, YAML, JSON |
| File Browser | Custom React component | Tree view of files in the terminal pod's workspace |
| Layout | Split panes (allotment) | Resizable panels for terminal, editor, and file tree |

#### UI Layout

```
┌─────────────────────────────────────────────────────┐
│  KFP Dashboard (existing)                           │
│  ┌───────────────────────────────────────────────┐  │
│  │  Runs / Pipelines / Experiments (existing)    │  │
│  │                                               │  │
│  │                                               │  │
│  └───────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────┐  │
│  │ ◉ Terminal  [+] Tab1  Tab2  Tab3    ▬ ☐ ✕    │  │
│  │ ┌──────────┬──────────────────────────────┐   │  │
│  │ │ Files    │  ~/workspace $ kubectl get   │   │  │
│  │ │ ├── src/ │  pods -n my-namespace        │   │  │
│  │ │ ├── data/│  NAME        READY  STATUS   │   │  │
│  │ │ └── ...  │  run-abc-1   1/1    Running  │   │  │
│  │ │          │  run-abc-2   0/1    Error    │   │  │
│  │ │          │  ~/workspace $               │   │  │
│  │ └──────────┴──────────────────────────────┘   │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

#### Contextual Entry Points

The terminal can be launched from multiple places in the existing UI:

- **Global**: A persistent "Terminal" button in the bottom status bar (collapsed by default).
- **Run Detail Page**: A "Debug in Terminal" action that opens a terminal pre-configured with:
  - `KFP_RUN_ID`, `KFP_PIPELINE_NAME` environment variables
  - Namespace set to the run's namespace
  - A helper script: `kfp-debug` that prints pod statuses and recent logs for the run.
- **Pipeline Detail Page**: An "Open Terminal" action scoped to the pipeline's namespace.

### Authentication and Authorization

The terminal server must enforce strict access controls:

1. **Authentication**: The terminal server sits behind the same Istio/OIDC gateway as the rest of Kubeflow. User identity is extracted from the `kubeflow-userid` header (set by the auth proxy).

2. **Namespace Authorization**: Before creating a terminal pod, the server verifies that the user has permission to create pods in the target namespace using a Kubernetes `SubjectAccessReview`:

```go
sar := &authorizationv1.SubjectAccessReview{
    Spec: authorizationv1.SubjectAccessReviewSpec{
        User: userID,
        ResourceAttributes: &authorizationv1.ResourceAttributes{
            Namespace: namespace,
            Verb:      "create",
            Resource:  "pods",
            Group:     "",
        },
    },
}
```

3. **ServiceAccount Scoping**: The terminal pod runs under the user's own ServiceAccount, inheriting only the RBAC permissions granted to that account. The terminal server itself does not grant elevated permissions.

4. **Network Policies**: Terminal pods should be subject to NetworkPolicies that restrict egress to the Kubernetes API server and explicitly allowed endpoints only.

### Session Management

| Parameter | Default | Description |
|-----------|---------|-------------|
| `maxSessionsPerUser` | 3 | Maximum concurrent terminal sessions per user |
| `idleTimeout` | 30m | Pod is terminated after this duration of inactivity |
| `maxSessionLifetime` | 8h | Absolute maximum lifetime for a terminal pod |
| `workspaceSize` | 1Gi | Size limit for the ephemeral workspace volume |

A background goroutine in the terminal server periodically checks for idle or expired sessions and deletes the corresponding pods. The `lastActive` timestamp is updated on every WebSocket message.

### Security Model

Security is a first-class concern for this feature:

1. **Pod Security**: Terminal pods run as non-root (UID 1000), with `runAsNonRoot: true`, no privilege escalation, and read-only root filesystem (with writable tmpfs for `/tmp` and workspace).

2. **Resource Limits**: Strict CPU/memory limits prevent terminal pods from starving pipeline workloads.

3. **No Host Access**: Terminal pods have no host mounts, no hostNetwork, and no hostPID.

4. **Audit Logging**: The terminal server logs all session creation/termination events and can optionally log terminal input for compliance (disabled by default).

5. **Image Pinning**: The terminal container image is versioned and pinned by digest in production deployments.

6. **Rate Limiting**: Session creation is rate-limited per user to prevent abuse.

## Test Plan

### Unit Tests
- Terminal server API handlers (session CRUD, auth checks).
- Session lifecycle management (creation, idle detection, cleanup).
- WebSocket message framing and relay.
- SubjectAccessReview integration logic.

### Integration Tests
- End-to-end session lifecycle: create session, connect WebSocket, execute command, verify output, disconnect, verify cleanup.
- Auth enforcement: verify that a user cannot create a terminal in a namespace they lack access to.
- Idle timeout: verify pod is garbage-collected after inactivity.
- Multi-tab: verify multiple WebSocket connections to the same pod work correctly.
- File browser: verify file listing and content read/write through the API.

### Frontend Tests
- xterm.js renders correctly and handles input/output.
- Terminal panel resize works.
- Tab management (create, switch, close).
- Contextual launch from run/pipeline detail pages passes correct parameters.

### Load Tests
- Simulate 50+ concurrent terminal sessions to verify resource consumption and stability.
- WebSocket connection stability under network disruption (reconnection logic).

## Migration Strategy

This is a new feature with no existing functionality to migrate from. Deployment is additive:

1. Deploy the `terminal-server` as a new Deployment in the `kubeflow` namespace.
2. Add the terminal server's Service to the Istio VirtualService routing rules.
3. Deploy the terminal container image to the cluster's container registry.
4. Enable the feature in the KFP frontend via a feature flag (`REACT_APP_TERMINAL_ENABLED=true`).

The feature is opt-in and does not affect existing KFP functionality when disabled.

## Frontend Considerations

- The terminal panel is a **new UI component** added to the KFP React frontend.
- It uses a **feature flag** so it can be disabled in environments where it is not desired (e.g., multi-tenant platforms with strict security policies).
- The panel is **lazy-loaded** to avoid increasing initial bundle size for users who don't use it.
- xterm.js (~200KB gzipped) and Monaco (~1.5MB gzipped) are loaded on-demand when the user first opens the terminal or editor.
- The terminal panel's state (open/closed, tab list) is persisted in `localStorage` so it survives page refreshes.

## KFP Local Considerations

This feature is **not applicable to KFP local execution**. The web terminal is inherently a cluster-connected feature. When running KFP locally, users already have direct terminal access on their machine. No changes to the KFP local experience are required.

## Implementation History

- 2026-02-21: Initial KEP draft proposed.

## Drawbacks

1. **Increased attack surface**: A web terminal is a powerful capability. Even with RBAC enforcement, a compromised user session could be used to interact with the cluster interactively. Mitigated by strict pod security, network policies, and audit logging.

2. **Resource overhead**: Each active terminal session consumes cluster resources (CPU, memory, a pod). In large multi-tenant deployments, this could add up. Mitigated by session limits, idle timeouts, and resource quotas.

3. **Maintenance burden**: A new microservice (`terminal-server`) and container image (`kfp-terminal`) must be maintained, versioned, and released alongside KFP. The terminal image must be kept up-to-date with `kubectl` and `kfp` CLI versions.

4. **Scope creep risk**: Users may expect this to evolve into a full IDE (Jupyter, VS Code Server). The non-goals section explicitly scopes this as a lightweight terminal, but the boundary will need to be maintained.

## Alternatives

### Alternative 1: Embed VS Code Server (code-server / OpenVSX)

Deploy a full [code-server](https://github.com/coder/code-server) instance per user.

**Benefits:**
- Full VS Code experience including extensions, IntelliSense, and integrated terminal.
- Mature, well-maintained open-source project.

**Downsides:**
- Significantly higher resource requirements per session (~1-2 GB RAM).
- Complex lifecycle management (persistent workspaces, extension sync).
- Overlaps with existing Kubeflow Notebook Server functionality.
- Overkill for the primary use case of quick command execution and debugging.

### Alternative 2: Link Out to Kubernetes Dashboard Terminal

Redirect users to the Kubernetes Dashboard's built-in `exec` feature.

**Benefits:**
- No new code to build or maintain.
- Already available in most clusters.

**Downsides:**
- Requires Kubernetes Dashboard to be deployed and accessible.
- No integration with KFP concepts (runs, pipelines, namespaces).
- No contextual pre-configuration (env vars, namespace, helper scripts).
- Poor UX: context-switching to a separate application.

### Alternative 3: CLI-Only Approach (Enhance `kfp` CLI)

Invest in the `kfp` CLI to provide better debugging and inspection commands locally.

**Benefits:**
- No new server-side components.
- Works offline and in air-gapped environments.

**Downsides:**
- Doesn't solve the context-switching problem.
- Requires users to have local `kubectl` and `kubeconfig` configured.
- Data scientists and ML engineers often prefer browser-based workflows.
