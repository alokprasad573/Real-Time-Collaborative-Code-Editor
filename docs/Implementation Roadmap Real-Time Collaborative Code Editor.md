# PRAXIS: Master Implementation Roadmap

Project: Real-Time Collaborative Code Editor | Author: Alok Prasad | Based on PRD v1.2 and ARCHITECTURE v1.2 | Status: Planning | 31 October 2026

Ye document do roadmaps ko merge karta hai: ChatGPT wala ROADMAP.md (13-phase structure, Phase 0, implementation rules, release milestones) aur chat mein bana week-by-week roadmap (tasks, exit criteria, PRD acceptance mapping, risks). Total: Phase 0 (pre-work) + 12 phases, 18 hafte, 2 releases.

## 1. Roadmap Overview

| Phase | Week | Implementation | Milestone | Release |
| --- | --- | --- | --- | --- |
| 0 | Pre-work (2-3 din) | Architecture validation, repo audit, traceability | Approved technical foundation | R1 |
| 1 | 1 | Setup, Docker, PostgreSQL, Redis, authentication | Stable application foundation | R1 |
| 2 | 2 | Rooms, roles, Monaco editor | Working IDE | R1 |
| 3 | 3-4 | Yjs, WebSockets, Hocuspocus, live cursors, persistence | Real-time collaboration | R1 |
| 4 | 5-6 | Execution queue, Docker sandbox, six languages | **Release 1: MVP** | R1 |
| 5 | 7-8 | Multi-file workspace, file tree, tabs, terminal | Complete workspace | R2 |
| 6 | 9 | Real-time text chat | Collaboration communication | R2 |
| 7 | 10-11 | GitHub integration, source control | Repository management | R2 |
| 8 | 12-13 | Language Server Protocol (LSP) | Intelligent editor | R2 |
| 9 | 14-15 | Debug Adapter Protocol (DAP) | Interactive debugging | R2 |
| 10 | 16 | WebRTC voice chat, TURN | Voice collaboration | R2 |
| 11 | 17 | Scaling, security, performance | Production hardening | R2 |
| 12 | 18 | Testing, deployment, documentation | **Release 2** | R2 |

### Core Collaboration Technologies

| Technology | Responsibility |
| --- | --- |
| WebSocket | Clients aur collaboration server ke beech messages ka transport |
| Yjs | Shared document data manage karna aur concurrent changes merge karna |
| Hocuspocus | Collaboration connections aur Yjs synchronization coordinate karna |

## 2. Implementation Rules

1. Phases defined sequence mein complete karo, jab tak koi dependency approved change na maange.
2. Ek time par ek phase par kaam karo.
3. Koi bhi file modify ya create karne se pehle existing code review karo.
4. Approved PRD aur architecture follow karo.
5. Approved technologies ko silently replace mat karo.
6. Har feature implement aur test karke hi complete mark karo.
7. Important technical decisions aur configuration changes document karo.
8. Agla phase tab tak shuru mat karo jab tak current phase ke exit criteria pass na ho jaayein.
9. Security, maintainability aur scalability poore development mein dhyan mein rakho.
10. Phase complete hone par roadmap update karo.
11. Har phase se pehle uske detailed tasks aur acceptance tests likh kar approve karo.
12. Scope badhe to PRD ke MoSCoW priorities follow karo, aur Could-have items sabse pehle kaato.

## 3. Phase Details

### Phase 0: Architecture Validation aur Repository Audit (pre-work)

**Objective:** Implementation shuru karne se pehle PRD, architecture, technical decisions aur repository structure validate karna.

**Tasks**

- PRD aur ARCHITECTURE.md review karo, aur ek requirements traceability matrix banao (har FR ID kis phase mein aayegi).
- Repository audit: agar pehle se code, dependencies ya config hai to list karo. Fresh repo hai to ye step skip karke structure define karo.
- Frontend, backend, database, Redis aur infrastructure ki responsibilities validate karo.
- Decisions finalize karo: Express vs Fastify, auth strategy (JWT + Redis session), Hocuspocus/WebSocket authentication, Yjs persistence (debounced PostgreSQL + Redis cache), GitHub App vs OAuth App.
- Repository structure define karo: `web`, `api`, `collab`, `worker`, `shared` (Zod schemas).
- Coding standards, environment variables aur testing conventions set karo.
- PRD ki purani line theek karo: section 17 mein 'single-file editing per document' likha hai, jo multi-file scope se match nahi karta.

**Deliverables:** Approved technical foundation, traceability document, finalized architecture decisions, repository structure aur implementation plan.

**Exit criteria:** Saare critical architecture decisions documented hain aur Phase 1 shuru karne ke liye ready hai.

### Phase 1: Project Setup aur Authentication (Week 1)

**Objective:** Ek stable, runnable application foundation banana.

**Tasks**

- Next.js, React aur TypeScript se frontend initialize karo; Tailwind CSS aur shadcn/ui configure karo.
- Node.js backend (JavaScript ES modules) initialize karo; API routing, Zod validation, Pino logging aur error handling set karo.
- ESLint, Prettier, JSDoc setup.
- Docker Compose: PostgreSQL aur Redis (MinIO baad mein).
- Prisma schema v1: `users`, `rooms`, `room_members`, `languages`, aur migrations.
- Redis connection management (ioredis).
- Auth: register, login, bcrypt/argon2 password hashing, JWT, auth middleware (FR-01, FR-02).
- Protected dashboard placeholder, health endpoints.
- Initial frontend, backend aur infrastructure tests; GitHub Actions CI skeleton (lint + test).

**Deliverables:** Running frontend aur backend, Docker-based dev environment, working registration/login, stable foundation.

**Exit criteria:** `docker compose up` se poora stack reliably start hota hai, migrations chalte hain, authentication kaam karta hai, foundational tests aur CI green hain.

### Phase 2: Room Management aur IDE (Week 2)

**Objective:** Initial collaborative coding environment banana.

**Tasks**

- `POST /rooms`, `GET /rooms/:id`, `POST /rooms/:id/join`, invite code aur invite link (FR-03, FR-04).
- Room ownership, membership aur owner/editor/viewer permissions.
- Next.js pages: login, dashboard, room page.
- shadcn/ui IDE-style layout (resizable panels, tabs).
- Monaco single-file editor, language selector, syntax highlighting (FR-16).
- Zustand store for UI state.

**Deliverables:** Working IDE with room management aur role-based access.

**Exit criteria:** Room banao, invite link bhejo, dusra user join kare, aur Monaco dono ke liye load ho. Role checks (owner/editor/viewer) API par enforce hon.

### Phase 3: Real-Time Collaboration (Weeks 3-4)

**Objective:** Multiple users ko ek hi editor mein collaborate karne dena.

**Tasks**

- Yjs CRDT integrate karo; Hocuspocus server WebSockets par connections aur Yjs sync coordinate kare.
- `onAuthenticate` mein JWT verify aur role check; viewers read-only (FR-15).
- `y-monaco` binding, awareness se live cursors aur selection (naam + colour), presence list (FR-12, FR-13).
- Persistence: debounced Yjs state `documents.yjs_state` (PostgreSQL) mein, Redis cache `doc:{id}:state`, restart ke baad recovery (FR-30).
- Reconnect aur offline edit sync (FR-14), awareness throttling.
- Custom Yjs test clients se concurrent editing aur convergence tests.
- Phase ke end mein Hocuspocus ka Redis extension laga kar 2 instances ka smoke test karo (full scaling Phase 11 mein).

**Deliverables:** Real-time collaborative editing with persistence.

**Exit criteria:** PRD acceptance criteria 1, 2 aur 6 pass (same line par concurrent edit ke baad identical documents, cursors sahi, server restart ke baad documents restore).

### Phase 4: Code Execution aur Sandboxing (Weeks 5-6) - Release 1 MVP

**Objective:** Users ko code securely execute karne dena.

**Tasks**

- `languages` table mein 6 rows seed karo; `POST /runs` (202 + runId) aur `GET /runs/:id`.
- BullMQ queue aur workers (dockerode); worker public API se alag chale, Docker socket kabhi sandbox mein mount na ho.
- Sandbox hardening: no network, non-root, read-only filesystem, saari capabilities dropped, no-new-privileges, memory/CPU/pids/time/output limits (PRD 10.1). Container har run ke baad destroy.
- Language images, is order mein: JavaScript, Python, C, C++, TypeScript, C#. .NET SDK image bada hai, isliye pre-pull aur warm container rakho. C/C++/C# ko compile ke liye lamba time limit.
- Output aur errors Redis ke through room ke saare users ko stream (FR-23), stdin support (FR-24).
- Per-user rate limit (FR-25), `executions` table aur run history (FR-26).
- Security test programs: infinite loop, fork bomb, memory bomb, network call, large output, file read/write attempts.

**Deliverables:** Saari six languages mein secure code execution.

**Exit criteria:** PRD acceptance criteria 3 aur 4 pass; warm worker par Run se first output p95 3 second se kam; security test programs se host par koi asar nahi.

**Milestone: RELEASE 1 (MVP).** Authenticated collaborative coding environment jisme rooms, real-time editing aur secure execution hai. Yahan release tag karo.

### Phase 5: Complete Workspace (Weeks 7-8)

**Objective:** Initial editor ko multi-file coding workspace mein badalna.

**Tasks**

- `files` table; file tree ek shared Yjs map, aur har file ka alag Yjs document (FR-42).
- react-arborist file explorer: create, rename, move, delete (FR-40, FR-17); editor tabs aur split view (FR-41).
- Path validation: `..`, absolute path aur symlink reject; workspace limits 200 files / 5 MB (FR-47).
- Project run: workspace snapshot S3/MinIO mein, container mein copy, compile (tsc, gcc, g++, dotnet build), phir run, entry file ke saath (FR-43).
- xterm.js terminal panel (FR-46).
- Global search/replace (FR-44); zip import/export, zip bomb limits ke saath (FR-45).

**Deliverables:** Complete multi-file workspace.

**Exit criteria:** PRD acceptance criterion 8: 3+ files ka project chosen entry file se saari 6 languages mein chalta hai, compile error aur missing entry file ke cases samet. Path traversal aur zip bomb tests pass.

### Phase 6: Real-Time Text Chat (Week 9)

**Objective:** Room participants ke beech communication.

**Tasks**

- Real-time messaging Hocuspocus ke WebSocket par, Redis channel `room:{id}:chat` se cross-server delivery (FR-90).
- `chat_messages` table, history load aur paging (`GET /rooms/:id/messages`) (FR-91).
- Chat UI: sender, timestamp, room membership aur permissions.
- Message 2000 characters tak, plain-text rendering, sanitising aur rate limit (FR-94).
- Owner dwara viewers ke liye chat allow/block (FR-93).
- Reconnect par delivery handle karna.
- Could-have: typing indicator, unread count, file:line reference, system messages (FR-92, FR-95, FR-96).

**Deliverables:** Real-time collaboration communication.

**Exit criteria:** Messages p95 300 ms ke andar sabko milte hain, refresh ke baad history aati hai (criterion 14 ka single-instance hissa; two-instance check Phase 11 mein).

### Phase 7: GitHub Integration aur Source Control (Weeks 10-11)

**Objective:** Workspaces ko GitHub repositories se jodna.

**Tasks**

- GitHub App (preferred) register karo; OAuth flow state parameter ke saath (CSRF se bachav) (FR-70).
- Token AES-256-GCM se encrypt karke `github_accounts` mein; token sirf Git service ke paas, kabhi browser, container ya logs mein nahi (FR-78).
- Repository import (public aur private) branch choose karke; isomorphic-git clone size/file-count limits ke saath, S3 mein working copy, Yjs docs mein load (FR-71).
- Source control panel: changed/added/deleted files, status aur diff merged Yjs state se, Monaco diff editor (FR-72, FR-73).
- Stage aur commit as the user, push, pull, branches (FR-74, FR-75); sirf editor/owner role aur connected GitHub account wale commit kar sakein (FR-79).
- Pull/merge ke time sabko notice; `.env` aur private key commit se pehle warning.
- Could-have (stretch): pull request creation aur merge conflict resolution (FR-76, FR-77).

**Deliverables:** Repository management aur source control.

**Exit criteria:** PRD acceptance criteria 11, 12 aur 13. Token kisi container, log ya browser response mein nahi hai.

### Phase 8: Language Server Protocol (Weeks 12-13)

**Objective:** Editor mein intelligent coding assistance.

**Tasks**

- LSP gateway: room + language ke hisaab se sandboxed container start, stdio aur WSS ke beech bridge.
- Redis registry `lsp:{room}:{lang}`, idle shutdown aur on-demand restart (FR-52); server unavailable ho to Monaco fallback bina editing toote (FR-53).
- `monaco-languageclient` integrate karo: diagnostics, autocomplete, hover, go to definition, find references, document sync (FR-50).
- Rollout order: TypeScript/JavaScript, Python, C/C++, C#.
- Per-room aur per-host container caps.
- Could-have: format document, rename symbol (FR-54).

**Deliverables:** Intelligent editor with language services.

**Exit criteria:** PRD acceptance criterion 9; warm server par autocomplete p95 300 ms se kam.

### Phase 9: Debug Adapter Protocol (Weeks 14-15)

**Objective:** Interactive debugging.

**Tasks**

- Debug gateway (DAP proxy) WebSocket par; sirf editor/owner session start karein, ek host controlling user.
- Language-specific adapters sandbox ke andar: js-debug, debugpy, gdb/lldb-dap, netcoredbg.
- Breakpoints gutter mein, Yjs ke through sab collaborators ke saath shared (FR-60).
- Continue, pause, step over, step into, step out (FR-62); variables, call stack, watch (FR-63).
- Viewers ko same state read-only dikhana (FR-64).
- Idle timeout aur maximum duration, sandbox isolation intact (FR-65).
- Rollout order: JS/TS, Python, C/C++, C#.
- Could-have: debug console (FR-66).

**Risk:** gdb/lldb ko `ptrace` chahiye, jabki sandbox mein saari capabilities drop hain. Custom seccomp profile ya sirf debug containers ke liye `SYS_PTRACE` plan karo, network off hi rahe. Ye C/C++ debugging ka sabse bada blocker ho sakta hai.

**Deliverables:** Interactive debugging environment.

**Exit criteria:** PRD acceptance criterion 10: breakpoint hit, step aur variables saari 6 languages mein, aur UI 500 ms ke andar update.

### Phase 10: WebRTC Voice Collaboration (Week 16)

**Objective:** Collaborators ke beech audio communication.

**Tasks**

- WebRTC signalling existing WebSocket par; audio-only.
- coturn deploy (public IP chahiye), STUN/TURN; `GET /voice/turn-credentials` short-lived credentials (FR-103).
- Room-based voice channel, peer-to-peer mesh, max 6 participants (FR-102).
- Join/leave handling, mute/unmute, speaking indicator, per-user volume, echo cancellation aur noise suppression (FR-100, FR-101, FR-105).
- Voice kabhi record ya store nahi hogi (FR-106).
- Could-have: owner dwara participant mute/remove (FR-104).

**Deliverables:** Voice collaboration.

**Exit criteria:** PRD acceptance criterion 15: 3+ users, mute aur speaking indicator, forced TURN relay mein bhi kaam kare.

### Phase 11: Scaling, Security aur Performance (Week 17)

**Objective:** Platform ko production workloads ke liye taiyaar karna.

**Tasks**

- 2+ Hocuspocus instances Redis Pub/Sub ke saath, Nginx sticky sessions; chat aur voice signalling ka cross-instance test.
- Authentication, authorization aur rate limiting strong karo (login, rooms, runs, chat, git).
- Sandbox isolation audit; gVisor evaluate; LSP/debug container caps.
- Poora security suite: path traversal, zip bomb, token leakage, chat injection, sandbox escape attempts.
- Database queries aur Redis usage optimize; Yjs garbage collection aur snapshot compaction.
- Monitoring: structured logs, health checks, metrics (connections, queue length, run duration).
- Load aur performance testing ki taiyaari.

**Deliverables:** Production-hardened application.

**Exit criteria:** PRD acceptance criterion 5 (do instances par users sync rahein), criterion 14 ka cross-instance hissa, criterion 13 dobara verify, aur security suite mein koi sandbox escape nahi.

### Phase 12: Testing, Deployment aur Documentation (Week 18)

**Objective:** Validation poora karke Release 2 ke liye taiyaar hona.

**Tasks**

- Unit aur integration tests complete (Jest, Supertest, Docker Compose); Playwright end-to-end (multi-browser editing, cursors, run output).
- Saari six languages validate karo; collaboration, chat, GitHub, LSP, DAP aur voice test karo; regression aur security testing.
- k6 load tests: 500 connections per server, 20 editors per room, p95 edit latency 200 ms se kam.
- Production Docker deployment (EC2/VPS), Nginx, HTTPS/TLS, health checks, auto restart, secrets.
- Full CI/CD: lint, type check, tests, image builds.
- README, architecture docs, deployment/operations docs, user aur developer documentation.

**Deliverables:** Tested application, production deployment, complete technical documentation, Release 2 readiness.

**Exit criteria:** PRD acceptance criterion 7 (CI pass, public URL, README) aur baaki saare criteria tick.

**Milestone: RELEASE 2.**

## 4. Release Milestones

| Release | Phases | Outcome |
| --- | --- | --- |
| Release 1: MVP | 0-4 (Week 0-6) | Authenticated collaborative coding environment: rooms, real-time editing aur secure code execution |
| Release 2 | 5-12 (Week 7-18) | Complete collaborative IDE: workspace, chat, GitHub, language servers, debugging, voice aur production hardening |

## 5. PRD Acceptance Criteria Mapping

| PRD criterion | Phase jahan pass hota hai |
| --- | --- |
| 1, 2, 6: identical documents, live cursors, restart recovery | Phase 3 |
| 3, 4: sandbox safety, run output sabko | Phase 4 |
| 5: do Hocuspocus instances sync mein | Phase 3 (smoke), Phase 11 (final) |
| 7: CI, public URL, README | Phase 12 |
| 8: multi-file run, 6 languages | Phase 5 |
| 9: autocomplete aur diagnostics | Phase 8 |
| 10: debugging, 6 languages | Phase 9 |
| 11, 12: GitHub import, commit, push, diff, pull | Phase 7 |
| 13: token leakage nahi | Phase 7 (build), Phase 11 (re-verify) |
| 14: chat delivery aur history | Phase 6, Phase 11 (two instances) |
| 15: voice, TURN fallback | Phase 10 |

## 6. Risks aur Mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Sandbox escape | Critical | No network, non-root, dropped capabilities, optional gVisor, alag worker host |
| GitHub token leakage | Critical | Encryption at rest, short-lived GitHub App tokens, tokens containers aur logs se bahar |
| Debugging complexity aur ptrace conflict | High | Ek language ek baar mein, standard adapters, custom seccomp ya targeted SYS\_PTRACE |
| Language server memory | High | Ek server per language per room, on-demand start, idle shutdown, host caps |
| Ek developer ke liye bada scope | High | Release 1 pehle, phase order follow karo, Could-have pehle kaato |
| Git operations aur live editing ka conflict | Medium | Merged Yjs state se commit, pull ke time sabko notice |
| Late scaling surprises | Medium | Redis multi-instance Phase 3 ke end mein smoke test, Phase 11 mein final |

**Scope cut order (agar time kam pade):** PR creation aur conflict resolution (FR-76/77), debug console (FR-66), format/rename (FR-54), chat extras (FR-92/95/96), voice moderation (FR-104), version snapshots (FR-31).

## 7. Project Completion Criteria

PRAXIS Release 2 ke liye ready tab hai jab:

- Saare phases (0-12) apne exit criteria pass kar chuke hon.
- Saari six languages mein intended execution, LSP aur debugging support ho.
- Collaborative editing aur persistence reliably kaam kare.
- Workspace, chat, GitHub aur voice features integrated hon.
- Security, performance aur regression testing complete ho.
- Production deployment aur documentation ready ho.

Note: ye roadmap project ka sequence aur intended outcomes define karta hai. Har phase ke detailed technical tasks, acceptance tests aur coding prompts us phase se pehle approve karke tay karo.
