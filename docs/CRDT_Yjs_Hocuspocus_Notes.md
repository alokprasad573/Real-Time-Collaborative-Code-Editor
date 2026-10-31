# CRDT + WebSockets + Yjs + Hocuspocus

## Learning Notes --- Lessons 1 to 12

> **Goal:** CRDT ko theory aur hands-on practice ke saath samajhna, phir
> collaborative features ko PRAXIS mein confidently design karna.
>
> **Learning style:** Concept → diagram → small experiment → self-check.
>
> **Suggested pace:** Har lesson ko ek sitting mein complete karna
> zaroori nahi. Pehle concept samjho, phir code run karo, phir apne
> words mein explain karo.

------------------------------------------------------------------------

## Learning map

``` mermaid
flowchart TD
    L1["L1 · WebSocket basics"] --> L2["L2 · Why CRDT?"]
    L2 --> L3["L3 · Yjs architecture"]
    L3 --> L4["L4 · Shared types"]
    L4 --> L5["L5 · Updates & state vectors"]
    L5 --> L6["L6 · Hocuspocus server"]
    L6 --> L7["L7 · Synchronization"]
    L7 --> L8["L8 · Awareness & presence"]
    L8 --> L9["L9 · Authentication & authorization"]
    L9 --> L10["L10 · Persistence"]
    L10 --> L11["L11 · Scaling & production"]
    L11 --> L12["L12 · PRAXIS architecture"]
```

## How to use these notes

-   [ ] Read the lesson's key idea.
-   [ ] Trace its Mermaid diagram.
-   [ ] Run or write the small experiment.
-   [ ] Answer the self-check before opening its answer.
-   [ ] Record anything unclear in your own learning log.

------------------------------------------------------------------------

# Lesson 1 --- WebSockets and real-time communication

### 1.1 HTTP versus WebSocket

**HTTP** commonly follows a request → response pattern. If a client
wants frequent updates, it may poll the server repeatedly.

**WebSocket** begins with an HTTP upgrade handshake. After the upgrade,
client and server can exchange messages over one persistent, full-duplex
connection.

  -----------------------------------------------------------------------
  HTTP polling                        WebSocket
  ----------------------------------- -----------------------------------
  Repeated requests may be needed     One connection can carry many
                                      messages

  Client often asks whether data      Either side can send a message
  changed                             

  Useful for ordinary                 Useful for low-latency, ongoing
  request/response APIs               communication
  -----------------------------------------------------------------------

### 1.2 Connection lifecycle

``` mermaid
sequenceDiagram
    participant B as Browser
    participant S as WebSocket Server
    B->>S: HTTP Upgrade request
    S-->>B: 101 Switching Protocols
    Note over B,S: WebSocket connection established
    B->>S: "Hello"
    S-->>B: "Message received"
    S-->>B: "Another server event"
    B->>S: Close frame
    S-->>B: Close response
```

### 1.3 Minimal Node.js experiment

Install the `ws` package:

``` bash
npm install ws
```

Create `server.js`:

``` js
const WebSocket = require("ws");

const server = new WebSocket.Server({ port: 8080 });

server.on("connection", (socket) => {
  console.log("Client connected");
  socket.send("Welcome!");

  socket.on("message", (message) => {
    console.log("Received:", message.toString());
    socket.send(`Server received: ${message}`);
  });

  socket.on("close", () => {
    console.log("Client disconnected");
  });
});

console.log("Listening on ws://localhost:8080");
```

In a browser console:

``` js
const socket = new WebSocket("ws://localhost:8080");

socket.onopen = () => socket.send("Hello from browser");
socket.onmessage = (event) => console.log(event.data);
socket.onclose = () => console.log("Disconnected");
```

**Important:** This is a raw WebSocket echo example. It is not yet Yjs
or Hocuspocus synchronization.

### Remember

WebSocket moves messages between peers and servers. It does not
understand whether two edits conflict or how a shared document should
merge.

**Question:** Does using WebSockets automatically make a collaborative
editor conflict-free?

**Answer:** No. WebSockets provide communication. A data synchronization
strategy such as Yjs/CRDT is needed to merge concurrent edits correctly.


------------------------------------------------------------------------

# Lesson 2 --- Why CRDT? Conflicts and convergence

### 2.1 The conflict problem

Suppose two users load the same value, `0`, while offline.

-   User A changes it to `1`.
-   User B independently changes it to `2`.
-   Both reconnect.

A naive "last update overwrites the previous value" strategy may discard
one user's work. The result depends on arrival order rather than a
well-defined merge rule.

``` mermaid
flowchart TD
    D["Shared starting value: 0"]
    D --> A["User A offline: set to 1"]
    D --> B["User B offline: set to 2"]
    A --> M["Both reconnect"]
    B --> M
    M --> Q{"Naive overwrite?"}
    Q -->|Yes| X["One update may be lost"]
    Q -->|CRDT merge| Y["Replicas converge using defined rules"]
```

### 2.2 What CRDT means

**CRDT = Conflict-free Replicated Data Type.**

A CRDT is a data structure designed so replicas can independently accept
updates and later merge them, under its assumptions, to reach a
consistent result without requiring a central lock for every edit.

Key ideas:

-   **Replica:** A copy of shared data held by a client or server.
-   **Concurrent update:** Updates made without one update causally
    following the other.
-   **Merge:** Combining information from replicas.
-   **Convergence:** Replicas that have received the same updates reach
    equivalent state.

### 2.3 Three properties to recognize

For many state-based CRDT designs, merge is based on a join operation
with these properties:

-   **Commutative:** `merge(A, B) = merge(B, A)`
-   **Associative:** `merge(merge(A, B), C) = merge(A, merge(B, C))`
-   **Idempotent:** `merge(A, A) = A`

These properties help make merge robust to message order, grouping, and
duplicate delivery. Operation-based CRDTs have their own delivery and
operation requirements; do not assume every CRDT uses the same
mechanism.

### Remember

CRDT does not mean "there can never be a product-level conflict." It
means the data type has defined rules for reconciling concurrent
updates. Your application still needs to decide what the edits mean.

**Question:** Why is "the latest message received wins" not a general
solution for collaborative editing?

**Answer:** Arrival order may not represent user intent or causal order,
and an overwrite can erase a concurrent change.

------------------------------------------------------------------------

# Lesson 3 --- Yjs architecture and Y.Doc


### 3.1 Main pieces

-   **Y.Doc:** The shared document container. It owns shared types and
    tracks document changes.
-   **Shared types:** Collaborative structures such as `Y.Text`,
    `Y.Map`, and `Y.Array`.
-   **Provider:** Connects a Y.Doc to a transport or synchronization
    service.
-   **Persistence adapter:** Saves and loads document data.

Yjs is the CRDT framework. A provider is how updates are exchanged;
Hocuspocus is one server-side option in the Yjs ecosystem.

``` mermaid
flowchart LR
    UI["Editor UI"] <--> Y["Y.Doc"]
    Y <--> T["Y.Text / Y.Map / Y.Array"]
    Y <--> P["Provider"]
    P <--> WS["WebSocket"]
    WS <--> H["Hocuspocus"]
    H <--> DB["Persistence adapter / storage"]
```

### 3.2 Create a document

``` bash
npm install yjs
```

``` js
import * as Y from "yjs";

const doc = new Y.Doc();
const title = doc.getText("title");

title.insert(0, "Hello Yjs");

console.log(title.toString());
```

The name `"title"` identifies a shared type within this document.
Another client using the same document identity and shared-type name can
synchronize with it through a compatible provider.

### 3.3 Local document versus remote synchronization

Creating `new Y.Doc()` does not connect to another browser. It creates a
local collaborative data model. A provider and a communication path are
needed to exchange updates with other replicas.

### Remember

Yjs and WebSockets are not competitors: - Yjs manages shared state and
updates. - WebSocket carries messages. - A provider connects the two.

**Question:** Does `new Y.Doc()` by itself connect two users?

**Answer:** No. It creates one local document. A provider/network
connection is needed to exchange updates.


------------------------------------------------------------------------

# Lesson 4 --- Yjs shared types: Y.Text, Y.Map, Y.Array

### 4.1 Y.Text

``` js
const doc = new Y.Doc();
const body = doc.getText("body");

body.insert(0, "Collaborative ");
body.insert(body.length, "editing");

console.log(body.toString());
```

For rich-text editors, the editor integration may use formatting
attributes and editor-specific bindings. Plain `Y.Text` is the starting
concept, not a complete rich-text editor by itself.

### 4.2 Y.Map

``` js
const doc = new Y.Doc();
const profile = doc.getMap("profile");

profile.set("theme", "dark");
profile.set("fontSize", 14);

console.log(profile.get("theme"));
```

### 4.3 Y.Array

``` js
const doc = new Y.Doc();
const tasks = doc.getArray("tasks");

tasks.push(["Understand CRDT"]);
tasks.push(["Try Yjs"]);

console.log(tasks.toArray());
```

``` mermaid
flowchart TD
    D["Y.Doc"]
    D --> T["Y.Text"]
    D --> M["Y.Map"]
    D --> A["Y.Array"]
    T --> T1["Document content"]
    M --> M1["Named fields"]
    A --> A1["Ordered items"]
```

### Design tip

Choose a shared type because it matches the data's meaning and update
behavior---not simply because it is convenient. Keep
application-specific validation and business rules outside the
assumption that CRDT merging validates data.

**Question:** Which Yjs shared type is the natural starting point for
collaborative text?

**Answer:** `Y.Text`.

------------------------------------------------------------------------

# Lesson 5 --- Yjs updates and state vectors

### 5.1 Update flow

A Yjs document can emit updates when it changes. Those updates can be
transmitted to another replica and applied there.

``` js
import * as Y from "yjs";

const docA = new Y.Doc();
const docB = new Y.Doc();

docA.on("update", (update) => {
  Y.applyUpdate(docB, update);
});

docA.getText("body").insert(0, "Hello");

console.log(docB.getText("body").toString());
```

This is a local in-memory demonstration of update propagation. It does
not use a network.

### 5.2 State vectors

A state vector summarizes what a Yjs document has observed from the
relevant client histories. It helps determine which updates another
replica is missing.

``` mermaid
sequenceDiagram
    participant A as Replica A
    participant B as Replica B
    A->>B: State vector A
    B->>A: State vector B
    Note over A,B: Each side compares known history
    A->>B: Missing update information
    B->>A: Missing update information
    Note over A,B: Both replicas synchronize
```

A common Yjs pattern is:

``` js
const stateVector = Y.encodeStateVector(docA);
const diff = Y.encodeStateAsUpdate(docA, stateVector);
```

The state vector is not the document itself. It is compact
synchronization metadata. The update contains document changes needed by
the receiver.

### 5.3 Important distinction

-   **Update:** Encoded change information.
-   **State vector:** Summary of what a document has seen.
-   **Snapshot:** A saved representation of document state, often used
    with an update history or persistence strategy.

Do not treat these terms as interchangeable.

**Question:** What is the main purpose of a Yjs state vector?

**Answer:** It summarizes the document's known client histories so
missing updates can be identified efficiently.

------------------------------------------------------------------------

# Lesson 6 --- Hocuspocus server architecture


### 6.1 What is Hocuspocus?

Hocuspocus is a Yjs-based backend framework for collaborative
applications. It provides a server that manages document connections and
synchronization, with hooks and integrations for authentication and
persistence.

A conceptual view:

``` mermaid
flowchart TD
    C1["Browser A · Y.Doc"] <-->|WebSocket| H["Hocuspocus server"]
    C2["Browser B · Y.Doc"] <-->|WebSocket| H
    C3["Browser C · Y.Doc"] <-->|WebSocket| H
    H --> AUTH["Authentication / access checks"]
    H --> SYNC["Document synchronization"]
    H --> STORE["Persistence integration"]
```

### 6.2 Document names

A client connects to a named document, for example:

-   `project-123`
-   `document-456`
-   `workspace-42:document-456`

The name is a routing/identity concept. It is not authorization. The
server must check whether the authenticated user is allowed to access
that document.

### 6.3 Minimal server shape

Hocuspocus APIs can evolve, so use the current official documentation
for exact package versions and configuration options. Conceptually, a
server setup looks like this:

``` js
import { Server } from "@hocuspocus/server";

const server = Server.configure({
  port: 1234,

  async onAuthenticate({ token, documentName }) {
    // Verify token and check document access here.
    // Reject unauthorized connections.
  },
});

server.listen();
```

This is an architectural illustration. Before running it, verify the
current Hocuspocus API and implement real authentication; do not deploy
an empty authentication hook as a security control.

### Remember

Hocuspocus is not the CRDT algorithm. Yjs provides the CRDT document
model; Hocuspocus hosts the collaborative server workflow.


**Question:** Does knowing a document name prove a user is allowed to
access it?

**Answer:** No. The server must authenticate the user and authorize
access to that specific document.


------------------------------------------------------------------------

# Lesson 7 --- WebSocket synchronization with Yjs and Hocuspocus

### 7.1 End-to-end sequence

``` mermaid
sequenceDiagram
    participant A as Browser A
    participant P as Yjs Provider
    participant H as Hocuspocus
    participant B as Browser B
    A->>P: Connect to document room
    P->>H: WebSocket connection + sync protocol
    H-->>P: Sync information / document state
    P-->>A: Apply remote updates
    B->>H: Connect to same authorized document
    H-->>B: Synchronize document state
    A->>P: Local Yjs update
    P->>H: Send encoded update
    H-->>B: Forward/synchronize update
    B->>B: Apply update to local Y.Doc
```

This is a conceptual flow. The actual Yjs synchronization protocol
includes message types and steps beyond this simplified sequence.

### 7.2 Browser-side concept

The Hocuspocus provider package connects a local Y.Doc to the server. A
typical setup uses:

``` js
import * as Y from "yjs";
import { HocuspocusProvider } from "@hocuspocus/provider";

const doc = new Y.Doc();

const provider = new HocuspocusProvider({
  url: "ws://localhost:1234",
  name: "project-123",
  document: doc,
});
```

The exact API and options should be checked against the installed
provider version.

### 7.3 What to verify in a two-client test

-   [ ] Both clients use the same server URL.
-   [ ] Both clients use the same document name.
-   [ ] Both clients are authorized.
-   [ ] Each client has its own `Y.Doc`.
-   [ ] Both connect through a compatible provider.
-   [ ] A change in one client appears in the other.
-   [ ] Disconnect/reconnect behavior is tested.

### Debugging checklist

If two users cannot see the same document, check in this order: 1. Is
the WebSocket connected? 2. Is the document name identical? 3. Did
authentication succeed? 4. Is the provider configured correctly? 5. Are
updates being emitted and applied? 6. Is the UI bound to the same shared
type that is changing?

**Question:** If the WebSocket says "connected," does that prove the two
Yjs documents have synchronized?

**Answer:** No. A live transport connection is only one requirement. The
Yjs sync protocol must complete and the correct document must be loaded.

------------------------------------------------------------------------

# Lesson 8 --- Awareness, presence, and cursors

### 8.1 Document data and presence are different

**Document data** includes content that should remain after a user
disconnects: text, tasks, structured project data.

**Awareness/presence** includes temporary information such as: - Who is
online. - Cursor position. - Current selection. - User display name and
cursor color.

Awareness is designed for ephemeral state. It is not a durable audit log
or a substitute for authentication.

``` mermaid
flowchart TD
    Y["Collaborative session"]
    Y --> DOC["Y.Doc · durable collaborative content"]
    Y --> AW["Awareness · temporary presence"]
    DOC --> PERSIST["Persist document state"]
    AW --> LIVE["Show online users / cursors"]
```

### 8.2 Awareness example concept

A client can publish user information to the awareness state:

``` js
provider.awareness.setLocalStateField("user", {
  name: "Asha",
  color: "#7C3AED",
});
```

Use only the fields supported by your editor integration. Never trust a
client-supplied name or role as proof of identity or permission.

### 8.3 Cursor rendering

An editor binding can translate awareness information into remote cursor
decorations. The exact implementation depends on the editor, such as
Tiptap/ProseMirror.

### Remember

A user leaving should normally remove their live presence, but their
document edits should remain.


**Question:** Should "User B is currently online" normally be stored as
permanent document content?

**Answer:** No. It is temporary presence information and belongs in
awareness/presence handling.


------------------------------------------------------------------------

# Lesson 9 --- Authentication and authorization


### 9.1 Two different questions

-   **Authentication:** Who is this user?
-   **Authorization:** Is this user allowed to access this document, and
    what may they do?

A valid login token does not automatically grant access to every
document.

``` mermaid
flowchart TD
    C["Client connects with token + document name"]
    C --> V["Verify token"]
    V -->|Invalid| DENY["Reject connection"]
    V -->|Valid| U["Identify user"]
    U --> A["Check access to requested document"]
    A -->|Denied| DENY
    A -->|Allowed| JOIN["Allow document session"]
```

### 9.2 Recommended authorization checks

For each connection: 1. Verify token signature, issuer, audience, and
expiry. 2. Identify the user from trusted token claims. 3. Resolve the
requested document to its project/workspace. 4. Check membership and
document-level permissions. 5. Apply read/write permissions. 6. Re-check
permissions where required by your revocation model.

### 9.3 Security mistakes to avoid

-   Trusting a user ID sent in the browser.
-   Assuming an obscure document name is secret.
-   Checking access only in the frontend.
-   Allowing every authenticated user into every room.
-   Logging raw access tokens.
-   Forgetting authorization for persistence APIs and exports.

### Remember

The server is the security boundary. Hiding a document in the UI is not
access control.

**Question:** A user has a valid login token. Can they automatically
open every document?

**Answer:** No. Authentication identifies the user; authorization checks
whether they may access the requested document.

------------------------------------------------------------------------

# Lesson 10 --- Persistence, loading, and reconnection

### 10.1 Lifecycle

``` mermaid
flowchart TD
    CON["Client requests document"]
    CON --> AUTH["Authenticate + authorize"]
    AUTH --> LOAD["Load stored Yjs state"]
    LOAD --> SYNC["Synchronize connected clients"]
    SYNC --> EDIT["Users edit collaboratively"]
    EDIT --> UPDATE["Receive Yjs updates"]
    UPDATE --> SAVE["Persist updates or encoded state"]
    SAVE --> RESTART["Server restarts / client reconnects"]
    RESTART --> LOAD
```

### 10.2 Persistence options

Common strategies include:

-   **Store updates:** Keep an append-only history of encoded updates,
    then load/replay them.
-   **Store snapshots/state:** Periodically save an encoded document
    state to reduce replay work.
-   **Hybrid:** Store updates and periodically compact into snapshots.

The right choice depends on document size, update volume, recovery
needs, and storage design.

### 10.3 Important operational concerns

-   Use stable document identifiers.
-   Avoid two writers overwriting each other's stored state.
-   Make persistence operations resilient to retries.
-   Plan snapshot/compaction.
-   Handle partially failed saves.
-   Test server restarts and reconnects.
-   Back up data and test restoration.

### 10.4 Persistence is not the same as presence

A disconnected user's document edits may already be persisted. Their
awareness state should not be treated as durable content.

**Question:** If a Hocuspocus server restarts, what makes document
content available again?

**Answer:** A persistence layer must load previously stored Yjs document
data. An in-memory document alone is lost when the process stops.


------------------------------------------------------------------------

# Lesson 11 --- Scaling and production architecture

### 11.1 Single server versus multiple servers

With one server, clients for a document can connect to the same process.
With multiple server instances, clients for the same document may land
on different processes.

``` mermaid
flowchart TD
    LB["Load balancer"]
    LB --> S1["Hocuspocus instance A"]
    LB --> S2["Hocuspocus instance B"]
    S1 --> C1["Clients connected to A"]
    S2 --> C2["Clients connected to B"]
    S1 <--> BUS["Cross-instance coordination / update propagation"]
    BUS <--> S2
    S1 --> DB["Shared persistence"]
    S2 --> DB
```

A shared database by itself does not necessarily provide live
cross-instance synchronization. You need an architecture that propagates
document updates between instances, using a supported coordination
mechanism or a deliberate routing strategy.

### 11.2 Production topics

-   **Routing:** Keep document connections coordinated; sticky sessions
    can help with connection routing but do not solve every
    cross-instance problem.
-   **Propagation:** Ensure updates reach clients connected to other
    instances.
-   **Persistence:** Prevent lost updates and unsafe concurrent writes.
-   **Resource limits:** Bound document size, connection count, message
    size, and update rates.
-   **Observability:** Track connection counts, sync failures, save
    latency, and memory use.
-   **Recovery:** Test restarts, network loss, duplicate messages, and
    storage outages.
-   **Security:** Re-check authorization and protect tokens and document
    data.

### 11.3 Questions to answer before scaling

-   How many concurrent connections are expected?
-   How large can a document become?
-   How many updates per second are expected?
-   What recovery point and recovery time are acceptable?
-   Can one document be active on multiple server instances?
-   How will a user's revoked access take effect?


**Question:** Does adding a load balancer and shared database
automatically synchronize live updates across Hocuspocus instances?

**Answer:** No. Cross-instance update propagation and document
coordination must be designed explicitly.


------------------------------------------------------------------------

# Lesson 12 --- Applying the architecture to PRAXIS

### Learning objectives


### 12.1 A possible PRAXIS architecture

Treat this as a starting proposal, not an assumption about the current
PRAXIS codebase. The actual design should be checked against the PRD and
repository.

``` mermaid
flowchart TB
    subgraph CLIENT["PRAXIS clients"]
        A["Browser A"]
        B["Browser B"]
        YA["Y.Doc + editor binding"]
        YB["Y.Doc + editor binding"]
        A --> YA
        B --> YB
    end

    subgraph BACKEND["Backend services"]
        API["PRAXIS API"]
        AUTH["Identity + permissions"]
        HP["Hocuspocus server"]
    end

    subgraph DATA["Data layer"]
        DB[("PostgreSQL")]
        REDIS[("Redis · optional coordination/cache")]
        STORE[("Yjs updates / snapshots")]
    end

    YA <-->|WebSocket provider| HP
    YB <-->|WebSocket provider| HP
    A -->|HTTPS API| API
    B -->|HTTPS API| API
    HP --> AUTH
    API --> AUTH
    AUTH --> DB
    HP <--> STORE
    HP <-.-> REDIS
    STORE --> DB
```

### 12.2 Responsibility boundaries

| Component | Responsibility |
|---|---|
| **PRAXIS frontend** | Editor UI, local `Y.Doc`, provider connection. |
| **PRAXIS API** | Normal application APIs and business workflows. |
| **Identity service** | Login, session management, and token issuance. |
| **Hocuspocus** | Collaborative document sessions and Yjs synchronization. |
| **Authorization layer** | Decide who can read or write each document. |
| **PostgreSQL or chosen storage** | Durable document data and application records. |
| **Redis (if needed)** | Coordination, caching, or supported scaling needs—not automatically the source of truth. |

### 12.3 Suggested implementation sequence

1.  Audit the existing PRAXIS document and permission model.
2.  Choose one small document type for the first collaboration proof of
    concept.
3.  Create two independent browser clients with separate Y.Docs.
4.  Connect both to a local Hocuspocus server.
5.  Verify simultaneous edits and offline/reconnect behavior.
6.  Add authentication and document-level authorization.
7.  Add persistence and restart/recovery tests.
8.  Integrate presence and editor cursors.
9.  Test failure cases and security boundaries.
10. Plan scaling only after the single-instance behavior is reliable.

### 12.4 Acceptance checklist

-   [ ] Two users can edit the same document.
-   [ ] Concurrent edits converge as expected.
-   [ ] Reconnecting clients receive missing changes.
-   [ ] Unauthorized users cannot read or write documents.
-   [ ] Server restart does not erase persisted content.
-   [ ] Presence disappears when a user disconnects.
-   [ ] Permission revocation behavior is defined and tested.
-   [ ] Updates are not lost during persistence failures.
-   [ ] Logs and metrics help diagnose connection and sync failures.


**Question:** Should PRAXIS implement multi-server scaling before
proving that two clients can reliably collaborate on one document?

**Answer:** No. First prove the core collaboration, security,
persistence, and recovery flow. Scale after the single-instance design
is understood and tested.

------------------------------------------------------------------------

# Final revision sheet

## The stack in one sentence

-   **WebSocket:** Provides the communication channel.
-   **Yjs:** Maintains shared CRDT data and merges updates.
-   **Hocuspocus:** Hosts Yjs collaboration sessions and server-side
    integrations.
-   **Awareness:** Carries temporary presence information.
-   **Persistence:** Makes document state survive disconnections and
    restarts.
-   **Authorization:** Controls who can access each document.

## Common misconceptions

## Common misconceptions

| Misconception | Correct understanding |
|---|---|
| **WebSockets solve conflicts** | WebSockets transport messages; Yjs handles CRDT merging. |
| **Hocuspocus is the CRDT** | Yjs provides the CRDT data model. |
| **Connected means synchronized** | Connection and document synchronization are distinct. |
| **Awareness is permanent data** | Awareness is ephemeral presence. |
| **A valid login means access to all docs** | Each document needs authorization. |
| **Shared DB solves multi-server sync** | Cross-instance propagation must also be designed. |
| **Saving only the latest UI state is always safe** | Persistence must preserve the intended Yjs update/state semantics. |

## Glossary

| Term | Meaning |
|---|---|
| **Replica** | A copy of a collaborative document. |
| **Concurrent** | Changes made without a causal ordering between them. |
| **Convergence** | Replicas reach an equivalent state after receiving the same updates. |
| **Y.Doc** | Yjs shared document container. |
| **Shared type** | A collaborative data structure, such as `Y.Text`. |
| **Provider** | Connects Yjs to a synchronization transport. |
| **State vector** | A summary of a document's known client histories. |
| **Awareness** | Temporary presence and cursor-related state. |
| **Persistence** | Durable storage of document state or updates. |
| **Authorization** | Permission check for a requested resource. |

## Official learning resources

Use these as the source of truth for current APIs and version-specific
setup:

-   [Yjs documentation](https://docs.yjs.dev/)
-   [Yjs GitHub repository](https://github.com/yjs/yjs)
-   [Hocuspocus
    documentation](https://tiptap.dev/docs/hocuspocus/introduction)
-   [Hocuspocus GitHub
    repository](https://github.com/ueberdosis/hocuspocus)
-   [MDN --- WebSocket
    API](https://developer.mozilla.org/en-US/docs/Web/API/WebSocket)
-   [Mermaid documentation](https://mermaid.js.org/)

> **Version note:** Package APIs and deployment guidance can change.
> Treat code snippets marked as conceptual as learning examples; verify
> exact APIs against the version you install.

------------------------------------------------------------------------

## Your progress tracker

-   [ ] L1 --- WebSocket fundamentals
-   [ ] L2 --- CRDT and convergence
-   [ ] L3 --- Yjs and Y.Doc
-   [ ] L4 --- Yjs shared types
-   [ ] L5 --- Updates and state vectors
-   [ ] L6 --- Hocuspocus server
-   [ ] L7 --- Synchronization
-   [ ] L8 --- Awareness and presence
-   [ ] L9 --- Authentication and authorization
-   [ ] L10 --- Persistence and recovery
-   [ ] L11 --- Production scaling
-   [ ] L12 --- PRAXIS architecture

**Recommended next step:** Complete L1 and L2 experiments first. Do not
start integrating CRDT into PRAXIS until you can explain the difference
between transport, synchronization, conflict resolution, and
persistence.
