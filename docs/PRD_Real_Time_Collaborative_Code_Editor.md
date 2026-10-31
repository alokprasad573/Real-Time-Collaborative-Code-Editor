# Project Requirements Document: Real-Time Collaborative Code Editor

Prepared by Alok Prasad | Version 1.2 | Status: Draft | Date: 31 October 2026

## 1. Document Control

| Version | Date | Author | Description |
|---|---|---|---|
| 1.0 | 28 Oct 2026 | Alok Prasad | Initial draft |
| 1.2 | 31 Oct 2026 | Alok Prasad | IDE features, GitHub, chat and voice added; six supported languages |

Related document: ARCHITECTURE.md (system architecture, database ER diagram, execution, CRDT, language server, debugging, GitHub and voice flow diagrams).

Version 1.2 (31 Oct 2026): debugging, language servers and multi-file projects moved into scope, GitHub integration, in-room text chat and voice chat added, the supported languages set to six (JavaScript, TypeScript, Python, C, C++ and C#, each with full run, language server and debugging support, matching ARCHITECTURE.md v1.2), and the project plan extended to 18 weeks in two releases. The frontend now includes shadcn/ui, and the backend uses JavaScript (ES modules) instead of TypeScript.

## 2. Introduction

### 2.1 Purpose

This document defines the requirements for a web-based real-time collaborative code editor. It describes what the system must do, the quality attributes it must meet, the technology choices with reasons, and how the project will be tested, deployed and delivered.

### 2.2 Product Overview

The product lets multiple users open the same code file in a shared room, edit it simultaneously with live cursors, and run the code safely inside isolated Docker containers with preset memory and time limits, in the same way coding platforms such as LeetCode and HackerRank do.

### 2.3 Intended Audience

Developers building the system, reviewers and recruiters evaluating the project, and anyone deploying or maintaining it.

### 2.4 Definitions

| Term | Meaning |
|---|---|
| CRDT | Conflict-free Replicated Data Type. A data structure that lets replicas be edited independently and always merge to the same result. |
| OT | Operational Transformation. An older approach that transforms concurrent operations against each other, usually with a central server. |
| Room | A shared workspace where invited users edit the same documents. |
| Awareness | Temporary, non-persistent state such as cursor position, selection and user colour. |
| Sandbox | An isolated, resource-limited container in which user code is executed. |

## 3. Problem Statement and Goals

### 3.1 Problem Statement

Remote pair programming, interview practice and teaching need a shared editor where everyone sees changes instantly and code can be run without installing anything. Simple shared editors lose edits when two people type at the same time, and running untrusted code on a server is dangerous if it is not isolated.

### 3.2 Goals

1. Allow several users to edit one file at the same time with no lost or conflicting edits.
2. Show every collaborator's cursor and selection live.
3. Execute code in any supported language safely, with hard resource limits.
4. Scale the real-time layer horizontally as the number of users grows.
5. Deliver a deployed, documented and tested project suitable for a resume and interviews.

### 3.3 Success Metrics (targets to validate with load tests)

| Metric | Target |
|---|---|
| Edit propagation latency (same region, p95) | Under 200 ms |
| Concurrent editors per room | At least 20 |
| Concurrent WebSocket connections per server instance | At least 500 |
| Time from Run click to first output (p95, warm worker) | Under 3 seconds |
| Document convergence under concurrent edit tests | 100 percent identical on all clients |
| Sandbox escape or host impact in security tests | Zero |

## 4. Users and Stakeholders

| Persona | Need |
|---|---|
| Student or learner | Code together with a friend or mentor and run the code instantly. |
| Interviewer and candidate | Shared live coding session with execution. |
| Developer pair | Quick collaborative debugging without setup. |
| Project owner and maintainer | Easy deployment, monitoring and safe execution. |

## 5. Scope

### 5.1 In Scope

- User registration and login, room creation and invite links.
- Real-time collaborative editing with CRDT, live cursors and presence.
- Multi-language code execution in Docker sandboxes: 6 languages (JavaScript, TypeScript, Python, C, C++ and C#), each with full IDE support (see section 6.9).
- Document persistence, version snapshots and run history.
- Horizontal scaling of Hocuspocus collaboration servers using Redis.
- Deployment, CI pipeline, tests and documentation.

Added in version 1.1 (VS Code style IDE features):

- Multi-file workspaces with a file tree, tabs, global search and project builds with an entry file.
- Language servers (LSP) for autocomplete, hover documentation, go to definition and diagnostics.
- Debugging with breakpoints, stepping, variables and call stack, using the Debug Adapter Protocol (DAP).
- GitHub integration: sign in with GitHub, import repositories, source control panel with diff, commit, push, pull, branches and pull requests.
- A terminal-style output panel for build and run output, plus in-room text chat and audio voice chat.

### 5.2 Out of Scope (version 1)

- Video calls and screen sharing (audio-only voice chat is in scope).
- Extension marketplace and compatibility with third-party VS Code extensions.
- Mobile native apps.
- Billing, teams and enterprise single sign-on.

## 6. Functional Requirements

Priority uses MoSCoW: M = Must have, S = Should have, C = Could have.

### 6.1 Authentication and Rooms

| ID | Requirement | Priority |
|---|---|---|
| FR-01 | Users can register and log in with email and password. | M |
| FR-02 | Passwords are stored only as salted hashes. | M |
| FR-03 | An authenticated user can create a room and receive a shareable invite link. | M |
| FR-04 | Users can join a room through an invite link, with a role of owner, editor or viewer. | M |
| FR-05 | Guest access to a room through a link, with a display name. | C |

### 6.2 Collaborative Editing

| ID | Requirement | Priority |
|---|---|---|
| FR-10 | Multiple users can edit the same document simultaneously. | M |
| FR-11 | Concurrent edits merge automatically using a CRDT with no lost updates. | M |
| FR-12 | Each user's cursor and selection are visible live, with name and colour. | M |
| FR-13 | A presence list shows who is currently in the room. | M |
| FR-14 | Edits made while briefly offline sync correctly after reconnect. | S |
| FR-15 | Viewers can see changes but cannot edit. | M |
| FR-16 | Syntax highlighting and language selection per file. | M |
| FR-17 | Users can create, rename and delete files inside a room. | S |

### 6.3 Code Execution

| ID | Requirement | Priority |
|---|---|---|
| FR-20 | A user can run the current file and see stdout, stderr and exit code. | M |
| FR-21 | Every run executes in a fresh, isolated Docker container that is destroyed afterwards. | M |
| FR-22 | Each run has preset limits for memory, CPU, execution time, processes and output size. | M |
| FR-23 | The run output is shown to all users in the room in real time. | S |
| FR-24 | Optional standard input can be supplied for a run. | S |
| FR-25 | Run requests are rate limited per user. | M |
| FR-26 | Run history (status, time, memory, output) is stored and viewable. | S |

### 6.4 Persistence and History

| ID | Requirement | Priority |
|---|---|---|
| FR-30 | Document state is saved automatically and restored when a room is reopened. | M |
| FR-31 | Users can create named version snapshots and restore them. | C |

### 6.5 Multi-file Workspace

| ID | Requirement | Priority |
|---|---|---|
| FR-40 | A file tree lets users create, rename, move and delete files and folders. | M |
| FR-41 | Several files can be open in tabs, with split view. | S |
| FR-42 | Every file is its own Yjs document, and the folder structure is a shared Yjs map, so file tree changes sync live between users. | M |
| FR-43 | A project run uses an entry file or a run configuration. The whole workspace is copied into the container, compiled if needed (tsc, gcc, g++, dotnet build), then run. | M |
| FR-44 | Global search and replace across all files. | S |
| FR-45 | Import and export a workspace as a zip file. | S |
| FR-46 | A terminal-style panel (xterm.js) shows build and run output. | S |
| FR-47 | Workspace limits are enforced (for example 200 files and 5 MB total) and unsafe paths such as .. and symlinks are rejected. | M |

### 6.6 Language Intelligence (Language Servers)

| ID | Requirement | Priority |
|---|---|---|
| FR-50 | Autocomplete, hover documentation, go to definition, find references and diagnostics for JavaScript, TypeScript, Python, C, C++ and C# through the Language Server Protocol. | S |
| FR-51 | Each language server runs in its own sandboxed container, one per language per room, shared by all users in the room. | S |
| FR-52 | Idle language servers shut down automatically and start again on demand. | M |
| FR-53 | If a language server is unavailable, the editor falls back to Monaco syntax highlighting without breaking editing. | M |
| FR-54 | Format document and rename symbol. | C |

### 6.7 Debugging

| ID | Requirement | Priority |
|---|---|---|
| FR-60 | Breakpoints can be set in the editor gutter and are shared with all collaborators. | S |
| FR-61 | A debug session can be started and stopped for JavaScript, TypeScript, Python, C, C++ and C#. | S |
| FR-62 | Continue, pause, step over, step into and step out. | S |
| FR-63 | Variables, call stack and watch expression panels. | S |
| FR-64 | One user hosts the session and controls execution, while other users can watch the same state. | S |
| FR-65 | Debug sessions run in the same sandbox restrictions as normal runs, with an idle timeout and a maximum duration. | M |
| FR-66 | A debug console for evaluating expressions. | C |

### 6.8 GitHub Integration

| ID | Requirement | Priority |
|---|---|---|
| FR-70 | Users can sign in with GitHub or connect a GitHub account to their profile. | S |
| FR-71 | A user can import a repository (public or private) into a workspace and choose a branch. | S |
| FR-72 | A source control panel lists changed, added and deleted files, like VS Code. | S |
| FR-73 | A side-by-side diff view for any changed file. | S |
| FR-74 | Stage files and commit with a message, authored as the user who commits. | S |
| FR-75 | Push, pull, create and switch branches. | S |
| FR-76 | Create a pull request from the workspace. | C |
| FR-77 | Detect merge conflicts and let users resolve them in the editor. | C |
| FR-78 | GitHub tokens are encrypted at rest, never sent to the browser and never available inside sandbox containers. | M |
| FR-79 | Only room members with the editor or owner role and a connected GitHub account can commit and push. | M |

### 6.9 Supported Languages

The system supports exactly six languages, and only these can be run: JavaScript, TypeScript, Python, C, C++ and C#. All six get the full feature set (run, language server and debugger). Monaco can still highlight other file types such as JSON or Markdown, but they cannot be run or debugged. Because the sandbox has no network, only the standard library and packages pre-installed in each language image are available (no npm install, pip install or NuGet restore at run time). C, C++ and C# need a compile step, so their containers get a longer build time limit, and the large .NET SDK image should be pre-pulled on execution hosts. Adding a language later needs only a registry entry (Docker image, file extension, compile command, run command and limits).

| Language | Docker image | Compile and run | Language server | Debugger |
|---|---|---|---|---|
| JavaScript | node (LTS) | node file.js | typescript-language-server | vscode-js-debug |
| TypeScript | node with tsc | tsc, then node | typescript-language-server | vscode-js-debug with source maps |
| Python | python 3 | python file.py | Pyright | debugpy |
| C | gcc | gcc with -g, then run | clangd | gdb (DAP mode) or lldb-dap |
| C++ | gcc (g++) | g++ with -g, then run | clangd | gdb (DAP mode) or lldb-dap |
| C# | .NET SDK | dotnet build, then dotnet run | csharp-ls (Roslyn based) | netcoredbg |

### 6.10 Chat and Voice Communication

| ID | Requirement | Priority |
|---|---|---|
| FR-90 | Room members can send and receive text messages in real time. | M |
| FR-91 | Messages are saved, and the recent history loads when a user joins, with paging for older messages. | S |
| FR-92 | Typing indicator, timestamps and unread message count. | C |
| FR-93 | The owner can allow or block chat for viewers. | S |
| FR-94 | Messages are sanitised, limited to 2000 characters and rate limited. | M |
| FR-95 | A message can reference a file and line range, and clicking it jumps to that code. | C |
| FR-96 | System messages appear in chat for events such as a user joining, a run finishing or a commit being pushed. | C |
| FR-100 | Users can join and leave a voice channel in a room (audio only). | S |
| FR-101 | Mute and unmute, a speaking indicator and per-user volume. | S |
| FR-102 | Voice uses WebRTC. Up to 6 people in one voice channel with peer-to-peer connections, signalling over the existing WebSocket. | S |
| FR-103 | A TURN server provides a fallback for restricted networks, with short-lived credentials. | S |
| FR-104 | The room owner can mute or remove a participant from voice. | C |
| FR-105 | Browser echo cancellation and noise suppression are enabled. | S |
| FR-106 | Voice audio is never recorded or stored. | M |

## 7. Non-Functional Requirements

| Category | Requirement |
|---|---|
| Performance | Edit propagation under 200 ms p95 in the same region. The editor must stay responsive with 20 active editors per document. |
| Scalability | Hocuspocus collaboration servers are stateless and scale horizontally behind a load balancer, with Redis sharing updates between instances. Execution workers scale independently through a queue. |
| Reliability | Documents survive server restarts. Clients reconnect automatically and resync. A crashed execution container never affects other users. |
| Security | Input validation, JWT authentication, role-based authorisation on every WebSocket and API action, HTTPS and WSS only, and a hardened sandbox (see section 10). |
| Availability | Target of 99 percent for the demo deployment, with health checks and automatic restart. |
| Usability | Works on current Chrome, Firefox, Edge and Safari. Responsive layout for laptop and tablet. |
| Maintainability | JavaScript (ES modules) on the backend and TypeScript on the frontend, ESLint and Prettier, JSDoc type hints on the backend, modular services and documented APIs. |
| Observability | Structured logs, health endpoints and basic metrics for connections, queue length and run durations. |

Additional non-functional requirements for version 1.1 (targets to validate):

| Area | Target |
|---|---|
| Language server response | Autocomplete under 300 ms p95 once the server is warm. |
| Debugger | Step and continue actions reflected in the UI within 500 ms. |
| Chat | Message delivery to all room members under 300 ms p95. |
| Voice | Audio latency under 400 ms for peer-to-peer connections, with a TURN fallback. |
| Git operations | Clone of a repository up to 100 MB within 60 seconds. |
| Resource control | A maximum number of language server and debug containers per room and per host, with idle shutdown. |

## 8. Technology Stack and Rationale

### 8.1 Frontend

| Technology | Purpose | Why this choice |
|---|---|---|
| Next.js (React) with TypeScript | Web application framework | Component-based UI, file-based routing, server-side rendering for fast first load and SEO of landing pages, and a very large community. TypeScript catches type errors early. Alternative considered: plain React with Vite, which has less built-in routing and SSR. |
| shadcn/ui (Radix UI primitives, lucide-react icons) | UI component library | Accessible components such as buttons, dialogs, tabs, dropdown and context menus, tooltips, toasts, command palette and resizable panels, built on Radix UI and styled with Tailwind CSS. The component code is copied into the project, so it is fully customisable, and tabs, resizable panels and menus suit an IDE-style layout. Alternatives: MUI or Chakra UI, which are heavier and harder to restyle. |
| Monaco Editor | Code editor component | It is the editor engine of VS Code, so it provides syntax highlighting, themes, autocomplete and multi-cursor out of the box. Alternative: CodeMirror 6, which is lighter but Monaco gives a more familiar IDE feel. |
| Yjs and y-monaco | CRDT document and editor binding | Yjs is a mature, fast CRDT library with compact binary updates, an awareness protocol for cursors and ready-made Monaco bindings. Alternative: Automerge, which is slower for large text. |
| Tailwind CSS | Styling | Fast, consistent UI development with small production CSS. It is also the styling layer used by shadcn/ui. |
| Zustand | Client state | Minimal, simple state management for UI state such as room and run output. Redux would be more boilerplate than needed. |

### 8.2 Backend

| Technology | Purpose | Why this choice |
|---|---|---|
| Node.js with JavaScript (ES modules) | Runtime | Event-driven, non-blocking I/O suits thousands of long-lived WebSocket connections, and plain JavaScript needs no build step, which keeps the backend simple and quick to develop. Trade-off: there is no compile-time type checking, so the backend relies on Zod validation, JSDoc type hints, ESLint and tests to catch mistakes. |
| Express.js (or Fastify) | REST API | Simple, well-known framework for auth, rooms and run requests. Fastify is an option if higher throughput is needed. |
| WebSocket, Yjs and Hocuspocus | WebSocket transports messages, Yjs manages shared document data and merges concurrent changes, and Hocuspocus coordinates connections and Yjs synchronization. | Hocuspocus provides a server-side collaboration layer for Yjs over WebSockets, including connection handling and synchronization. This keeps the transport, shared data model and collaboration coordination responsibilities distinct. |
| BullMQ | Job queue for code execution | Reliable Redis-backed queue with retries, concurrency control and rate limiting, so heavy executions never block editing traffic. |
| dockerode | Docker control from Node.js | Programmatic creation, limiting and removal of containers through the Docker Engine API. |
| Zod | Request validation | Runtime schema validation for all API and WebSocket inputs. This matters more in JavaScript because there is no compile-time type check. |
| JWT with bcrypt (or argon2) | Authentication | Stateless tokens suit horizontally scaled servers, and bcrypt or argon2 provides safe password hashing. |
| Pino | Logging | Fast structured JSON logging. |

### 8.3 Databases and Caching

| Technology | Purpose | Why this choice |
|---|---|---|
| PostgreSQL | Primary database | Relational data (users, rooms, members, documents, runs) with foreign keys and transactions. It stores Yjs binary state in bytea columns and has strong JSON support. Alternative: MongoDB, which is flexible but this data is clearly relational. |
| Prisma ORM | Database access | A generated client with autocomplete, schema migrations and safe parameterised queries, and it works well from plain JavaScript. |
| Redis (ioredis) | Pub/Sub, presence, cache, queue backend, rate limits | In-memory speed. Pub/Sub lets several Hocuspocus collaboration servers share document updates, keys with TTL suit presence and sessions, and it is also the backend for BullMQ. |

### 8.4 Code Execution Sandbox

| Technology | Purpose | Why this choice |
|---|---|---|
| Docker | Isolated execution per run | Per-run containers with limits for CPU, memory, processes and time, and a clean filesystem every time. Used by coding platforms for the same reason. |
| Language images (node, python, gcc, .NET SDK) | Runtimes | Official minimal images are maintained and reproducible. |
| gVisor (runsc) as optional hardening | Extra kernel isolation | Containers share the host kernel, so gVisor adds a user-space kernel layer that reduces kernel exploit risk. |

### 8.5 DevOps, Testing and Tooling

| Technology | Purpose | Why this choice |
|---|---|---|
| Docker Compose | Local development | One command starts the app, Postgres, Redis and workers. |
| Nginx (or AWS ALB) | Reverse proxy and load balancer | TLS termination, WebSocket proxying and sticky routing. |
| GitHub Actions | CI pipeline | Runs lint, the frontend type check and tests on every push. |
| AWS EC2 (free tier) or any VPS | Deployment | Docker is required on the host for sandbox execution, which rules out serverless platforms for the worker. |
| Jest and Supertest | Unit and API tests | Standard for Node.js services. |
| Playwright | End-to-end tests | Simulates multiple browsers editing one document. |
| k6 | Load testing | Measures WebSocket latency and connection capacity to produce real numbers for the resume. |
| ESLint and Prettier | Code quality | Consistent style and early error detection. |

### 8.6 Why CRDT (Yjs) Instead of OT

| Aspect | OT | CRDT (chosen) |
|---|---|---|
| Server role | Needs a central server to order and transform operations | Servers only relay updates, which is simpler and scales better |
| Offline editing | Difficult | Natural: replicas merge on reconnect |
| Implementation complexity | High, transformation functions are error prone | Library handles it (Yjs) |
| Cost | Low metadata | Some metadata overhead, reduced by Yjs compact encoding and garbage collection |

### 8.7 Additional Technologies for Version 1.1

| Technology | Purpose | Why this choice |
|---|---|---|
| monaco-languageclient and vscode-ws-jsonrpc | Connect Monaco to language servers over WebSocket | Lets Monaco speak the Language Server Protocol, the same standard VS Code uses, so one client works for every language. Alternative: CodeMirror LSP client, but Monaco is already the editor. |
| typescript-language-server, Pyright, clangd, csharp-ls | Language servers for JavaScript, TypeScript, Python, C, C++ and C# | These are the standard, actively maintained servers behind VS Code language support. Each runs in its own sandboxed container. |
| Debug Adapter Protocol (DAP) with vscode-js-debug, debugpy, gdb DAP mode, lldb-dap or netcoredbg | Debugging for JavaScript, TypeScript, Python, C, C++ and C# | DAP is the standard VS Code uses, so one generic debug panel works with every adapter. Adapters run inside the sandbox, never on the host. |
| Custom Node.js debug gateway | Proxy between the browser and the debug adapter | Forwards DAP messages over a WebSocket, checks permissions and enforces session time limits. |
| xterm.js | Terminal-style output panel | The same terminal component VS Code uses. |
| react-arborist | File tree | Virtualised tree with drag and drop and keyboard support, suitable for large projects. |
| Monaco diff editor | Side-by-side diff for source control | Built into Monaco, no extra dependency. |
| Octokit (@octokit/rest, @octokit/auth-app) | GitHub REST API client | Official client for repositories, branches, pull requests and user data. |
| GitHub App (preferred) or OAuth App | GitHub authentication | A GitHub App gives fine-grained per-repository permissions and short-lived tokens. An OAuth App is simpler but has broader scopes. |
| isomorphic-git | Clone, status, diff, commit, push and pull in Node.js | Pure JavaScript Git that runs without a Git binary on the host. Alternative: simple-git, which needs Git installed. |
| Node crypto (AES-256-GCM) | Encrypt GitHub tokens at rest | Built in, authenticated encryption, with the key kept in a secret manager or environment variable. |
| S3-compatible object storage (AWS S3 or MinIO) | Workspace zip files, cloned working copies and run snapshots | Cheap storage for large files that should not live in PostgreSQL. |
| Native WebRTC API | Voice chat | Built into all modern browsers with echo cancellation and noise suppression. Peer-to-peer mesh is enough for up to 6 users, with no media server cost. |
| coturn | STUN and TURN server | Open source standard for NAT traversal and relay when direct connections fail. |
| LiveKit (optional, later) | Media server (SFU) | Open source option if voice rooms need to grow beyond what a peer-to-peer mesh can handle. |
| DOMPurify | Sanitising chat content | Prevents script injection if messages are ever rendered as rich text. Plain text rendering is the default. |

## 9. System Architecture Summary

The full diagrams are in ARCHITECTURE.md. In summary:

1. The Next.js client connects to a load balancer over HTTPS for REST calls and over WSS for Yjs updates.
2. The REST API handles authentication, rooms, files and run requests.
3. A cluster of stateless Hocuspocus collaboration servers coordinates connections and relays Yjs synchronization over WebSockets. Redis Pub/Sub shares updates between instances so users on different servers stay in sync.
4. Document state is saved to PostgreSQL in debounced snapshots, not on every keystroke.
5. A run request is placed on a Redis-backed queue. Execution workers pick up jobs, start an isolated Docker container, collect the output, destroy the container, and publish the result back to the room.

Version 1.1 adds these services beside the original components:

- **Workspace service:** the file tree is a shared Yjs map and every file is its own Yjs document. Text is stored in PostgreSQL, and zip files and cloned working copies in object storage.
- **Language server gateway:** starts one sandboxed language server container per language per room, and bridges it to the browser over a WebSocket. Idle servers are stopped.
- **Debug gateway:** starts the debug adapter inside a sandbox container and forwards DAP messages between the browser and the adapter.
- **Git service:** performs clone, status, diff, commit, push and pull for a workspace, and calls the GitHub API with encrypted tokens that never enter a sandbox.
- **Chat service:** part of the Hocuspocus collaboration servers. Messages are published through Redis to all servers and saved to PostgreSQL.
- **Voice signalling and TURN:** WebRTC offers, answers and ICE candidates travel over the existing WebSocket, and a coturn server relays audio when a direct connection is not possible.

## 10. Security Requirements

### 10.1 Sandbox Hardening

- No network access inside the container.
- Non-root user, read-only root filesystem and a small writable temporary directory.
- All Linux capabilities dropped and no-new-privileges enabled.
- Memory limit (for example 128 MB), CPU limit (for example 0.5 core), process limit and hard timeout (for example 5 seconds).
- Output size cap to prevent log flooding.
- Container is removed after every run, so no data carries over.
- Optional gVisor runtime for stronger kernel isolation.
- The Docker socket is never mounted into a sandbox container, and the worker runs separately from the public API.

### 10.2 Application Security

- JWT verification during the WebSocket handshake and role checks for every action.
- Input validation with Zod, parameterised queries through Prisma, and output escaping in the UI.
- Rate limiting for login, room creation and run requests.
- CORS restrictions, secure headers and HTTPS and WSS only.
- Secrets in environment variables and never committed to the repository.

### 10.3 Security for IDE, GitHub and Communication Features

- Language server and debug containers use the same restrictions as run containers: no network, non-root, read-only filesystem where possible, resource limits and idle shutdown.
- GitHub tokens are encrypted with AES-256-GCM, never returned to the browser, never placed in container environment variables, and never written to logs.
- The GitHub OAuth flow uses a state parameter to prevent CSRF, and the app requests only the minimum permissions.
- File paths are normalised and any path with .. , an absolute path or a symlink outside the workspace is rejected.
- Zip import and repository clone enforce size, file count and depth limits to stop zip bombs and oversized repositories.
- A warning is shown before committing files such as .env or private keys.
- Debug sessions can only be started by editors and owners, have an idle timeout and a maximum duration, and cannot reach the network.
- Chat messages are rendered as plain text, length limited and rate limited. Voice TURN credentials are short-lived and generated on the server.

## 11. Interface Overview

### 11.1 REST API (indicative)

| Method and path | Purpose |
|---|---|
| POST /auth/register, POST /auth/login | Create account and obtain a token |
| POST /rooms, GET /rooms/:id | Create and fetch a room |
| POST /rooms/:id/join | Join using an invite code |
| GET, POST, PATCH, DELETE /rooms/:id/documents | Manage files in a room |
| POST /runs | Submit a code run (returns 202 and a run id) |
| GET /runs/:id | Fetch run status and output |

### 11.2 WebSocket Channels

| Channel | Purpose |
|---|---|
| /collab/:documentId | Yjs sync and awareness (cursors, presence) |
| Run events | Run status and output pushed to all users in the room |

### 11.3 Additional Endpoints and Channels (Version 1.1)

| Endpoint or channel | Purpose |
|---|---|
| GET, POST, PATCH, DELETE /rooms/:id/files | File tree operations: create, rename, move and delete files and folders |
| POST /runs (with entryFile and language) | Run a multi-file project |
| GET /auth/github and GET /auth/github/callback | Connect a GitHub account |
| GET /github/repos | List the repositories the user can access |
| POST /rooms/:id/git/clone | Import a repository into the workspace |
| GET /rooms/:id/git/status and /git/diff | Changed files and diffs |
| POST /rooms/:id/git/commit, /push, /pull, /branch, /pr | Commit, push, pull, create a branch or open a pull request |
| GET /rooms/:id/messages | Chat history with paging |
| GET /voice/turn-credentials | Short-lived TURN credentials |
| WebSocket /lsp/:roomId/:language | Language server messages |
| WebSocket /debug/:sessionId | Debug adapter messages |
| WebSocket chat and voice events | Chat messages, typing events and WebRTC signalling |

## 12. Data Requirements

The database has these entities: users, rooms, room_members, files (the workspace tree), documents (one per file, with the latest Yjs state), document_snapshots, languages (the language registry), executions, github_accounts, room_repositories, debug_sessions and chat_messages. The relationships and fields are shown in the ER diagram in ARCHITECTURE.md. Redis holds temporary data only: pub/sub channels, presence hashes, cached document state, the execution queue, rate limit counters, sessions, the registry of running language server and debug containers, and voice channel membership.

## 13. Testing Strategy

| Level | What is tested | Tools |
|---|---|---|
| Unit | Services, validators, permission checks | Jest |
| Integration | REST API with Postgres and Redis, queue and worker flow | Supertest, Docker Compose |
| Concurrency | Several simulated clients typing at once, convergence of final documents | Custom Yjs test clients |
| End-to-end | Two or more browsers editing, cursors visible, run output shown | Playwright |
| Security | Infinite loop, fork bomb, memory bomb, large output, network call, file read and write attempts | Custom test programs run in the sandbox |
| Load | WebSocket connections, message latency, run queue behaviour | k6 |

Additional tests for version 1.1:

- Language server tests for completion, hover and diagnostics in each of the six languages.
- Debugger tests that set a breakpoint, hit it, step and read variables for JavaScript, TypeScript, Python, C, C++ and C#.
- Multi-file run tests, including compile errors and a missing entry file.
- Git tests against a test GitHub repository: clone, edit, commit, push, pull and conflict handling.
- Security tests for path traversal, zip bombs, token leakage into containers or logs, and chat injection.
- Chat tests for ordering, history and delivery across two server instances.
- Voice tests with two to six browsers, including a forced TURN relay.

## 14. Deployment Requirements

- Containerised services: web, API, Hocuspocus collaboration server, execution worker, PostgreSQL and Redis.
- Docker Compose for local development and a single-host or small multi-host deployment on a VPS or AWS EC2.
- Nginx as reverse proxy with TLS certificates and WebSocket upgrade support.
- GitHub Actions pipeline for lint, tests and image builds.
- Environment-based configuration with separate development and production settings.
- Health check endpoints and automatic container restart.

Additional deployment components for version 1.1: object storage (S3 or MinIO), a coturn server with a public IP, language server and debug container images for the six languages, and GitHub App credentials stored as secrets.

## 15. Project Plan and Milestones

| Phase | Deliverables | Estimate |
|---|---|---|
| 1. Foundation | Repository, project setup (ES modules, ESLint, Prettier), Docker Compose, PostgreSQL and Redis, authentication | Week 1 |
| 2. Rooms and editor | Room management, Monaco editor, Next.js pages | Week 2 |
| 3. Real-time collaboration | WebSocket transport, Yjs shared data and merging, Hocuspocus connection coordination and synchronization, live cursors, presence, persistence | Weeks 3 and 4 |
| 4. Code execution (Release 1, MVP) | Queue, workers, Docker sandbox, limits, run history, output streaming for JavaScript, TypeScript, Python, C, C++ and C# | Weeks 5 and 6 |
| 5. Multi-file workspace | File tree, tabs, project runs with an entry file, search, zip import and export, terminal panel | Weeks 7 and 8 |
| 6. In-room text chat | Chat service, history, unread count, system messages | Week 9 |
| 7. GitHub integration | GitHub sign-in, clone, source control panel, diff, commit, push, pull, branches, pull requests | Weeks 10 and 11 |
| 8. Language servers | Language server gateway for JavaScript, TypeScript, Python, C, C++ and C# | Weeks 12 and 13 |
| 9. Debugging | Debug gateway, breakpoints, stepping, variables and call stack for all six languages | Weeks 14 and 15 |
| 10. Voice chat | WebRTC signalling, coturn, mute, speaking indicator | Week 16 |
| 11. Scaling and hardening | Redis Pub/Sub across servers, rate limits, security tests | Week 17 |
| 12. Testing and deployment (Release 2) | End-to-end and load tests, CI, deployment, README and architecture docs | Week 18 |

## 16. Risks and Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Sandbox escape by malicious code | Critical | Strict container limits, no network, non-root, dropped capabilities, optional gVisor, separate worker host |
| Resource exhaustion from many runs | High | Queue with concurrency limits, per-user rate limits, hard timeouts |
| CRDT document growth over time | Medium | Yjs garbage collection, periodic snapshot compaction |
| Hocuspocus collaboration server failure | Medium | Stateless servers, automatic reconnect, state restored from Redis and PostgreSQL |
| Cursor or update flooding | Medium | Throttle awareness updates and limit message size |
| Scope growth | Medium | MoSCoW priorities and a fixed version 1 scope |

Additional risks for version 1.1:

| Risk | Impact | Mitigation |
|---|---|---|
| Debugging is complex and adds attack surface | High | Roll out one language at a time, reuse the standard debug adapters, keep the same sandbox limits, allow one controlling user per session |
| Language servers use a lot of memory | High | One server per language per room, start on demand, idle shutdown, caps per host |
| GitHub token leakage | Critical | Encryption at rest, GitHub App with short-lived tokens, tokens kept out of containers and logs, audit logging |
| GitHub API rate limits | Medium | Installation tokens, caching and conditional requests |
| Git operations conflict with live editing | Medium | Commit from the merged Yjs state, show a notice to all users during pull or merge, resolve conflicts in the editor |
| Voice quality and connectivity | Medium | TURN fallback, limit of 6 users per channel, optional media server later |
| Scope is large for one developer | High | Ship Release 1 (MVP) first, deliver features in the phase order above, and cut Could-have items first |

## 17. Assumptions and Constraints

- Users have a modern browser and a stable internet connection.
- The execution host has Docker installed and enough CPU and memory for parallel runs.
- Version 1 targets single-file editing per document and a small to medium user base.
- The free-tier or low-cost hosting available may limit load test numbers.

## 18. Acceptance Criteria

1. Two or more users editing the same line at the same time end with identical documents.
2. Live cursors and presence are visible with the correct name and colour.
3. Infinite loop, fork bomb, memory bomb and network access attempts are stopped by the sandbox and the host remains unaffected.
4. Run output appears to all users in the room.
5. With two Hocuspocus collaboration server instances running, users connected to different instances still stay in sync over WebSockets.
6. Documents are restored after a server restart.
7. CI passes, the application is deployed on a public URL, and the README contains the architecture and setup instructions.

Additional acceptance criteria for version 1.1:

8. A multi-file project with at least three files runs from a chosen entry file in JavaScript, TypeScript, Python, C, C++ and C#.
9. Autocomplete and error diagnostics appear in JavaScript, TypeScript, Python, C, C++ and C# files.
10. A breakpoint set by one user is hit, and variables, call stack and stepping work in all six languages.
11. A user can import a public and a private GitHub repository, edit with others, commit and push as themselves, and see the commit on GitHub.
12. The diff view shows the changes, and pulls and branch switches work without losing collaborative edits.
13. GitHub tokens are not present in any container, log or browser response.
14. Chat messages reach all members within the target latency across two server instances, and history loads after a refresh.
15. Voice chat works between at least three users, with mute and speaking indicators, and falls back to TURN when needed.

## 19. Future Enhancements

- Pull request review comments inside the editor.
- Additional languages such as Java and Go, added through the language registry.
- Voice recording and transcription, video calls and screen sharing.
- Interview mode with timers and question banks.
- Kubernetes-based auto scaling of execution workers.
- Persistent version history with diff view.