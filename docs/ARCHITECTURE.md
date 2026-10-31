# Real-Time Collaborative Code Editor: Architecture (v1.2)

**Core stack:** Next.js (React, shadcn/ui, Tailwind CSS), Node.js (JavaScript), WebSockets, Yjs (CRDT), Hocuspocus, Redis, PostgreSQL, Docker
**IDE features:** Monaco, Language Server Protocol, Debug Adapter Protocol, GitHub (Octokit, isomorphic-git), WebRTC voice, text chat

---

## 1. System Architecture

```mermaid
flowchart TB
    subgraph Client["Client (Browser)"]
        UI["Next.js App (React, shadcn/ui)"]
        Monaco["Monaco Editor"]
        YjsClient["Yjs Client (CRDT) + Awareness (live cursors)"]
        Panels["IDE Panels: File Tree, Source Control, Debug, Terminal, Chat"]
        LSPClient["monaco-languageclient (LSP client)"]
        RTC["WebRTC (voice, audio only)"]
        UI --> Monaco
        UI --> Panels
        Monaco <--> YjsClient
        Monaco <--> LSPClient
    end

    LB["Load Balancer (Nginx / AWS ALB, sticky sessions)"]

    subgraph Backend["Backend (Node.js)"]
        API["REST API: Auth, Rooms, Workspace Files, Run Requests, Chat History"]
        subgraph WSCluster["Hocuspocus Collaboration Cluster (stateless)"]
            WS1["Hocuspocus Server 1: connections, Yjs sync, chat, voice signalling"]
            WS2["Hocuspocus Server 2: connections, Yjs sync, chat, voice signalling"]
            WS3["Hocuspocus Server N: connections, Yjs sync, chat, voice signalling"]
        end
        LSPGW["LSP Gateway"]
        DBGGW["Debug Gateway (DAP proxy)"]
        GitSvc["Git Service (isomorphic-git + Octokit)"]
    end

    Redis[("Redis: Pub/Sub, Presence, Cache, Job Queue, Rate Limits, Container Registry")]
    PG[("PostgreSQL: Users, Rooms, Files, Documents, Runs, Chat, GitHub links")]
    S3[("Object Storage (S3 / MinIO): zips, cloned working copies, run snapshots")]

    subgraph ExecLayer["Sandboxed Container Layer (Docker)"]
        Worker["Execution Workers (Queue Consumers)"]
        RunC["Run containers: JavaScript, TypeScript, Python, C, C++, C#"]
        LSC["Language server containers: ts-language-server, Pyright, clangd, csharp-ls"]
        DAC["Debug adapter containers: js-debug, debugpy, gdb / lldb-dap, netcoredbg"]
        Worker --> RunC
    end

    GitHub["GitHub (REST API and git remote)"]
    TURN["coturn (STUN / TURN)"]

    UI -- "HTTPS (REST)" --> LB
    YjsClient -- "WebSocket (Yjs sync, chat, signalling)" --> LB
    LSPClient -- "WSS (LSP)" --> LB
    Panels -- "WSS (DAP)" --> LB
    LB --> API
    LB --> WS1
    LB --> WS2
    LB --> WS3
    LB --> LSPGW
    LB --> DBGGW

    WS1 <--> Redis
    WS2 <--> Redis
    WS3 <--> Redis

    API --> PG
    API --> Redis
    API --> GitSvc
    WS1 -- "debounced snapshots, chat messages" --> PG
    WS2 -- "debounced snapshots, chat messages" --> PG
    WS3 -- "debounced snapshots, chat messages" --> PG

    API -- "enqueue run job" --> Redis
    Redis -- "dequeue job" --> Worker
    Worker -- "store result" --> PG
    Worker -- "publish output" --> Redis
    Worker -- "workspace copy" --> S3

    LSPGW -- "start / stop, bridge stdio" --> LSC
    DBGGW -- "start / stop, bridge DAP" --> DAC
    LSPGW --> Redis
    DBGGW --> Redis

    GitSvc -- "clone, push, pull, PRs" --> GitHub
    GitSvc -- "working copies" --> S3
    GitSvc -- "encrypted tokens" --> PG

    RTC -- "audio, peer to peer" --> RTC
    RTC -- "relay fallback" --> TURN
```

### Code Execution Flow (multi-file project)

```mermaid
sequenceDiagram
    actor User
    participant FE as Next.js Frontend
    participant API as Node.js API
    participant Q as Redis Queue
    participant W as Execution Worker
    participant D as Docker Sandbox
    participant WS as Hocuspocus Collaboration Server

    User->>FE: Click "Run" (entry file, language)
    FE->>API: POST /runs (roomId, entryFile, language)
    API->>API: Auth, role and rate limit check
    API->>Q: Enqueue job (workspace snapshot)
    API-->>FE: 202 Accepted (runId)
    Q->>W: Dequeue job
    W->>D: Start container from language image (no network, CPU, memory, time limits)
    W->>D: Copy workspace, compile if needed (tsc, gcc, g++, dotnet build)
    D-->>W: stdout / stderr / exit code
    W->>D: Destroy container
    W->>WS: Publish result via Redis
    WS-->>FE: Push output to all users in the room (terminal panel)
```

### Concurrent Edit Flow (CRDT)

```mermaid
sequenceDiagram
    actor A as User A
    actor B as User B
    participant WS1 as Hocuspocus Server 1
    participant R as Redis Pub/Sub
    participant WS2 as Hocuspocus Server 2

    A->>WS1: Yjs update (insert "x")
    B->>WS2: Yjs update (insert "y")
    WS1->>R: Share Yjs update (room id)
    WS2->>R: Share Yjs update (room id)
    R-->>WS2: Update from A
    R-->>WS1: Update from B
    WS1-->>A: Deliver B's update
    WS2-->>B: Deliver A's update
    Note over A,B: CRDT merges both edits, same final document on both clients
```

### Language Server Flow (LSP)

```mermaid
sequenceDiagram
    actor User
    participant Mon as Monaco + languageclient
    participant GW as LSP Gateway
    participant R as Redis
    participant LS as Language Server Container

    User->>Mon: Open a Python / C / C++ / C# / TS / JS file
    Mon->>GW: WSS connect (roomId, language)
    GW->>R: Is a server already running for this room and language?
    alt not running
        GW->>LS: Start sandboxed container (no network, limits)
    end
    GW-->>Mon: Connected
    Mon->>GW: initialize, didOpen, completion request
    GW->>LS: Forward JSON-RPC over stdio
    LS-->>GW: Completion list, diagnostics
    GW-->>Mon: Forward response
    Note over GW,LS: Idle servers are stopped after a timeout
```

### Debugging Flow (DAP)

```mermaid
sequenceDiagram
    actor Host as Host user
    actor Viewer as Other collaborators
    participant FE as Debug UI
    participant DG as Debug Gateway
    participant DA as Debug Adapter Container

    Host->>FE: Start debugging (entry file, breakpoints)
    FE->>DG: WSS connect + start session (editor or owner role required)
    DG->>DA: Start sandboxed container with debug adapter
    DG-->>FE: Session ready
    FE->>DG: setBreakpoints, launch
    DG->>DA: DAP requests
    DA-->>DG: stopped event (breakpoint hit)
    DG-->>FE: Stopped, stack, variables
    DG-->>Viewer: Same state (read only view)
    Host->>FE: Step over / continue
    FE->>DG: DAP request
    DG->>DA: Forward
    Note over DG,DA: Idle timeout and maximum duration enforced, container destroyed at the end
```

### GitHub Flow (import, commit, push)

```mermaid
sequenceDiagram
    actor User
    participant FE as Next.js Frontend
    participant API as Node.js API
    participant GS as Git Service
    participant GH as GitHub
    participant PG as PostgreSQL
    participant S3 as Object Storage

    User->>FE: Connect GitHub
    FE->>GH: OAuth / GitHub App authorisation
    GH-->>API: Callback with code
    API->>GH: Exchange for token
    API->>PG: Store token (AES-256-GCM encrypted)
    User->>FE: Import repository (branch)
    FE->>API: POST /rooms/:id/git/clone
    API->>GS: Clone using user's token
    GS->>GH: git clone
    GS->>S3: Save working copy
    GS-->>API: File list loaded into workspace (Yjs docs created)
    User->>FE: Edit files together, open Source Control panel
    FE->>API: GET /git/status and /git/diff (from merged Yjs state)
    User->>FE: Commit and push
    FE->>API: POST /git/commit then /git/push
    API->>GS: Commit as the user, push
    GS->>GH: git push
    GH-->>FE: Commit visible on GitHub (system message in chat)
    Note over PG,GS: Tokens never reach the browser or any sandbox container
```

### Voice Chat Flow (WebRTC)

```mermaid
sequenceDiagram
    actor A as User A
    actor B as User B
    participant WS as Hocuspocus Collaboration Server (WebSocket signalling)
    participant API as Node.js API
    participant T as coturn (STUN / TURN)

    A->>API: GET /voice/turn-credentials
    API-->>A: Short-lived credentials
    A->>WS: Join voice channel (roomId)
    B->>WS: Join voice channel (roomId)
    WS-->>A: B joined
    A->>WS: WebRTC offer for B
    WS-->>B: Offer
    B->>WS: Answer
    WS-->>A: Answer
    A->>T: ICE candidates (STUN)
    A-->>B: Audio stream, peer to peer
    Note over A,T: If a direct connection fails, audio is relayed through TURN
```

---

## 2. Database Architecture

```mermaid
erDiagram
    USERS ||--o{ ROOMS : owns
    USERS ||--o{ ROOM_MEMBERS : joins
    ROOMS ||--o{ ROOM_MEMBERS : has
    ROOMS ||--o{ FILES : workspace
    FILES ||--o{ FILES : contains
    FILES ||--o| DOCUMENTS : has_content
    DOCUMENTS ||--o{ DOCUMENT_SNAPSHOTS : versions
    LANGUAGES ||--o{ FILES : typed_as
    LANGUAGES ||--o{ EXECUTIONS : used_by
    LANGUAGES ||--o{ DEBUG_SESSIONS : used_by
    ROOMS ||--o{ EXECUTIONS : runs
    FILES ||--o{ EXECUTIONS : entry_file
    USERS ||--o{ EXECUTIONS : triggers
    ROOMS ||--o{ DEBUG_SESSIONS : hosts
    USERS ||--o{ DEBUG_SESSIONS : starts
    USERS ||--o| GITHUB_ACCOUNTS : connects
    ROOMS ||--o| ROOM_REPOSITORIES : linked_to
    USERS ||--o{ ROOM_REPOSITORIES : connected_by
    ROOMS ||--o{ CHAT_MESSAGES : has
    USERS ||--o{ CHAT_MESSAGES : sends

    USERS {
        uuid id PK
        string email UK
        string name
        string password_hash
        string avatar_url
        timestamp created_at
    }

    ROOMS {
        uuid id PK
        uuid owner_id FK
        string name
        string invite_code UK
        boolean is_public
        boolean chat_enabled_for_viewers
        timestamp created_at
    }

    ROOM_MEMBERS {
        uuid id PK
        uuid room_id FK
        uuid user_id FK
        string role "owner, editor, viewer"
        timestamp joined_at
    }

    LANGUAGES {
        int id PK
        string name UK "javascript, typescript, python, c, cpp, csharp"
        string file_extension
        string docker_image
        string compile_cmd
        string run_cmd
        string lsp_server "typescript-language-server, pyright, clangd, csharp-ls"
        string debug_adapter "js-debug, debugpy, gdb or lldb-dap, netcoredbg"
        int memory_limit_mb
        int time_limit_ms
        boolean enabled
    }

    FILES {
        uuid id PK
        uuid room_id FK
        uuid parent_id FK
        string name
        string path
        string type "file or folder"
        int language_id FK
        timestamp created_at
        timestamp updated_at
    }

    DOCUMENTS {
        uuid id PK
        uuid file_id FK
        bytea yjs_state "latest merged Yjs document state"
        timestamp created_at
        timestamp updated_at
    }

    DOCUMENT_SNAPSHOTS {
        uuid id PK
        uuid document_id FK
        bytea yjs_state
        uuid created_by FK
        string label
        timestamp created_at
    }

    EXECUTIONS {
        uuid id PK
        uuid room_id FK
        uuid user_id FK
        int language_id FK
        uuid entry_file_id FK
        string workspace_snapshot_key "object storage key"
        text stdin
        text stdout
        text stderr
        int exit_code
        string status "queued, running, success, error, timeout"
        int time_ms
        int memory_kb
        timestamp created_at
        timestamp finished_at
    }

    DEBUG_SESSIONS {
        uuid id PK
        uuid room_id FK
        uuid host_user_id FK
        int language_id FK
        string status "starting, running, stopped, timeout"
        string end_reason
        timestamp started_at
        timestamp ended_at
    }

    GITHUB_ACCOUNTS {
        uuid id PK
        uuid user_id FK
        bigint github_user_id
        string github_login
        bytea access_token_encrypted "AES-256-GCM"
        string scopes
        timestamp token_expires_at
        timestamp connected_at
    }

    ROOM_REPOSITORIES {
        uuid id PK
        uuid room_id FK
        uuid connected_by FK
        string repo_full_name
        string default_branch
        string current_branch
        string last_synced_sha
        string working_copy_key "object storage key"
        timestamp cloned_at
    }

    CHAT_MESSAGES {
        uuid id PK
        uuid room_id FK
        uuid user_id FK
        string type "user or system"
        string body "max 2000 characters"
        string ref_file_path "optional code reference"
        int ref_line_start
        int ref_line_end
        timestamp created_at
    }
```

### Redis Data Structures (not tables, in-memory)

```mermaid
flowchart LR
    Redis[("Redis")]

    Redis --> K1["room:{roomId}:updates (Pub/Sub: Yjs updates between WS servers)"]
    Redis --> K2["room:{roomId}:presence (Hash: userId to cursor, name, color, TTL)"]
    Redis --> K3["doc:{docId}:state (String: cached Yjs state for fast load)"]
    Redis --> K4["queue:executions (BullMQ queue: pending run jobs)"]
    Redis --> K5["ratelimit:{userId}:{action} (Counter with TTL: run, chat and git limits)"]
    Redis --> K6["session:{token} (String: auth session, TTL)"]
    Redis --> K7["room:{roomId}:chat (Pub/Sub: chat messages and typing events)"]
    Redis --> K8["lsp:{roomId}:{language} (Hash: container id, last used, TTL for idle shutdown)"]
    Redis --> K9["debug:{sessionId} (Hash: container id, host user, state, TTL)"]
    Redis --> K10["voice:{roomId}:members (Set: users in the voice channel)"]
```

---

## 3. Supported Languages

**Total: 6 languages.** Only these six can be run, and all six get the full IDE feature set (run, language server and debugger). Monaco can still highlight other file types, but they cannot be run or debugged.

| Language | Docker image | Compile / run | Language server | Debugger |
|---|---|---|---|---|
| JavaScript | node (LTS) | node file.js | typescript-language-server | vscode-js-debug |
| TypeScript | node with tsc | tsc, then node (or ts-node) | typescript-language-server | vscode-js-debug with source maps |
| Python | python 3 | python file.py | Pyright | debugpy |
| C | gcc | gcc with -g, then run | clangd | gdb (DAP mode) or lldb-dap |
| C++ | gcc (g++) | g++ with -g, then run | clangd | gdb (DAP mode) or lldb-dap |
| C# | .NET SDK (mcr.microsoft.com/dotnet/sdk) | dotnet build, then dotnet run | csharp-ls (Roslyn based) or OmniSharp | netcoredbg |

Adding another language later only needs a new row in the `LANGUAGES` table plus its container images.

---

## Design Notes

- **CRDT (Yjs):** edits merge automatically, so there is no central locking and offline edits sync after reconnect. Every file is its own Yjs document, and the folder tree is a shared Yjs map.
- **Collaboration layer:** WebSockets transport messages, Yjs manages shared documents and merges concurrent changes, and Hocuspocus coordinates connections and Yjs synchronization. Stateless Hocuspocus servers scale behind the load balancer; Redis Pub/Sub keeps instances in sync for edits, chat and voice signalling.
- **Debounced persistence:** the Yjs state is saved to PostgreSQL every few seconds, not on every keystroke.
- **Isolated execution:** each run uses a short-lived container (no network, read-only filesystem, non-root, CPU / memory / time limits) that is destroyed after the run.
- **Language servers and debuggers:** use the same standards as VS Code (LSP and DAP), run in sandboxed containers started on demand, are shared by the room, and stop after an idle timeout.
- **GitHub:** tokens are encrypted at rest and stay on the Git service. They are never sent to the browser or any sandbox container. Commits are made from the merged Yjs state and authored as the user who commits.
- **Chat and voice:** chat travels over the existing WebSocket and is stored in PostgreSQL. Voice is audio only over WebRTC (peer to peer for up to 6 users, TURN fallback) and is never recorded.
- **C# and C/C++ builds:** these need a compile step before running, so their run containers get a longer build time limit than interpreted languages. The .NET SDK image is large (several hundred MB), so it should be pre-pulled on execution hosts and warm containers kept ready to avoid slow first runs.
- **No network in the sandbox:** only the standard library and packages pre-installed in each language image are available, so there is no npm install, pip install or NuGet restore at run time. Commonly used packages should be baked into the images.
- **Separate scaling:** execution workers, language server containers and debug containers scale independently from the editing servers.