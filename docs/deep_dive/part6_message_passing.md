# Part 6: Message Passing Layer — The Notification Backbone

## Overview

The message passing layer is the **lowest-level communication** in LoLa. But here's the key insight:

> **Message passing does NOT carry data. It carries notifications.**

The actual data goes through shared memory (Part 3). Message passing only tells the other side: "hey, something happened — go check shared memory."

---

## What Message Passing Does vs What Shared Memory Does

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  SHARED MEMORY (Parts 3-5)           MESSAGE PASSING (This Part)       │
│  ═════════════════════════           ════════════════════════════       │
│                                                                        │
│  • Carries actual DATA               • Carries NOTIFICATIONS           │
│  • Large payloads (KB-MB)            • Tiny payloads (bytes)           │
│  • Zero-copy, no kernel              • Goes through kernel (socket)    │
│  • Lock-free atomics                 • Blocking/async I/O              │
│                                                                        │
│  "Here's 4KB of camera frame"        "New frame ready in slot 3"      │
│  "Here's the temperature value"      "Event published!"               │
│  "Here's the method return"          "Method call request!"           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Architecture: N-to-1 Client-Server Pattern

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│   CLIENT A (Proxy)     CLIENT B (Proxy)     CLIENT C (Proxy)           │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐            │
│   │ ClientConn   │    │ ClientConn   │    │ ClientConn   │            │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘            │
│          │                    │                    │                    │
│          │    SEND/REQUEST    │                    │                    │
│          └────────────────────┼────────────────────┘                    │
│                               │                                         │
│                               ▼                                         │
│                    ┌──────────────────────┐                             │
│                    │      SERVER          │                             │
│                    │   (Skeleton side)    │                             │
│                    │                      │                             │
│                    │  ServerConnection A  │  ← per-client state         │
│                    │  ServerConnection B  │                             │
│                    │  ServerConnection C  │                             │
│                    └──────────────────────┘                             │
│                               │                                         │
│                    REPLY / NOTIFY                                       │
│                               │                                         │
│          ┌────────────────────┼────────────────────┐                    │
│          │                    │                    │                    │
│          ▼                    ▼                    ▼                    │
│   CLIENT A              CLIENT B              CLIENT C                 │
│                                                                         │
│   PROPERTIES:                                                           │
│   • Multiple clients → single server (n:1)                             │
│   • Each client has its own bidirectional session                       │
│   • Messages from same client: ordered                                 │
│   • Messages from different clients: NOT ordered                       │
│   • Server knows client identity (PID/UID/GID)                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## The 4 Message Types

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  CLIENT → SERVER                        SERVER → CLIENT                 │
│                                                                         │
│  ┌─────────────────────────┐           ┌─────────────────────────┐    │
│  │  SEND                   │           │  REPLY                  │    │
│  │                         │           │                         │    │
│  │  Fire-and-forget        │           │  Response to REQUEST    │    │
│  │  "Event published!"     │           │  "Here's your answer"   │    │
│  │  No response expected   │           │  Paired with REQUEST    │    │
│  └─────────────────────────┘           └─────────────────────────┘    │
│                                                                         │
│  ┌─────────────────────────┐           ┌─────────────────────────┐    │
│  │  REQUEST                │           │  NOTIFY                 │    │
│  │                         │           │                         │    │
│  │  RPC: expects REPLY     │           │  Async server push      │    │
│  │  "Call this method"     │           │  "Data ready in slot N" │    │
│  │  Blocks until response  │           │  Fire-and-forget        │    │
│  └─────────────────────────┘           └─────────────────────────┘    │
│                                                                         │
│  Binary protocol: [1-byte type code][payload bytes...]                 │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### How LoLa Uses Each Type

| Message Type | Used For |
|-------------|----------|
| **SEND** | Event notification: "skeleton published new data" |
| **REQUEST** | Method call: "proxy wants to invoke a method" |
| **REPLY** | Method response: "skeleton returns result" |
| **NOTIFY** | Event wake-up: "new data available, check SHM" |

> **Related file**: [score/message_passing/client_server_communication.h](../../score/message_passing/client_server_communication.h)

---

## Key Interfaces

### IClientConnection (Proxy side)

```cpp
class IClientConnection {
  public:
    // Fire-and-forget notification
    Result<void> Send(span<const uint8_t> message);
    
    // Synchronous RPC (blocks until reply)
    Result<span<const uint8_t>> SendWaitReply(span<const uint8_t> message);
    
    // Async RPC with callback
    Result<void> SendWithCallback(span<const uint8_t> message, ReplyCallback callback);
    
    // Connect to server asynchronously
    Result<void> Start(StateCallback state_cb, NotifyCallback notify_cb);
    
    // Disconnect
    Result<void> Stop();
};
```

**State Machine:**
```
    Start()              Connected              Stop()
STOPPED ──────► STARTING ──────► READY ──────► STOPPING ──► STOPPED
                                   ▲                │
                                   └── reconnect ───┘
```

### IServer (Skeleton side)

```cpp
class IServer {
  public:
    Result<void> StartListening(
        ConnectCallback on_connect,           // New client connected
        DisconnectCallback on_disconnect,     // Client disconnected
        SentCallback on_sent,                 // SEND received
        SentWithReplyCallback on_request      // REQUEST received (must reply)
    );
};
```

### IServerConnection (Per-client session)

```cpp
class IServerConnection {
  public:
    // Reply to a REQUEST
    Result<void> Reply(span<const uint8_t> reply_data);
    
    // Push notification to this client
    Result<void> Notify(span<const uint8_t> notification);
    
    // Client identity (PID, UID, GID)
    ClientIdentity GetClientIdentity() const;
    
    // Graceful disconnect
    void RequestDisconnect();
};
```

> **Related files**:
> - [score/message_passing/i_client_connection.h](../../score/message_passing/i_client_connection.h)
> - [score/message_passing/i_server.h](../../score/message_passing/i_server.h)
> - [score/message_passing/i_server_connection.h](../../score/message_passing/i_server_connection.h)

---

## Dual Transport: Linux vs QNX

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  SAME INTERFACE (IClientConnection, IServer)                            │
│                     │                                                   │
│          ┌──────────┴──────────┐                                       │
│          │                     │                                        │
│  ┌───────▼────────┐    ┌──────▼──────────┐                            │
│  │ Unix Domain    │    │  QNX Dispatch   │                            │
│  │ Engine         │    │  Engine         │                            │
│  │                │    │                 │                            │
│  │ • SOCK_STREAM  │    │ • QNX message   │                            │
│  │ • poll() loop  │    │   passing       │                            │
│  │ • Filesystem   │    │ • dispatch_     │                            │
│  │   path binding │    │   create()      │                            │
│  │ • SO_PEERCRED  │    │ • Pulse-based   │                            │
│  │   for identity │    │   dispatch      │                            │
│  │                │    │ • Resource mgr  │                            │
│  │ Linux-only     │    │   interface     │                            │
│  │                │    │                 │                            │
│  │                │    │ QNX-only        │                            │
│  └────────────────┘    └─────────────────┘                            │
│                                                                         │
│  Selected at COMPILE TIME via engine.h                                 │
│  Your code never changes!                                              │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Related files**:
> - [score/message_passing/unix_domain/](../../score/message_passing/unix_domain/) — Linux implementation
> - [score/message_passing/qnx_dispatch/](../../score/message_passing/qnx_dispatch/) — QNX implementation
> - [score/message_passing/engine.h](../../score/message_passing/engine.h) — Compile-time selection

---

## How Events Use Message Passing

```
SKELETON (publishes event)                PROXY (subscribed)
══════════════════════════                ═════════════════

1. Write data to SHM slot
2. Mark slot READY (atomic)
3. Send NOTIFY to all 
   subscribed proxy connections
         │                                      │
         │   ┌─────────────────────┐           │
         └──►│ Message Passing:    │           │
             │ NOTIFY "slot 2 rdy" │──────────►│
             └─────────────────────┘           │
                                               ▼
                                    4. NotifyCallback fires
                                    5. Proxy calls GetNewSamples()
                                    6. Reads slot 2 from SHM
                                       (zero-copy)
```

---

## How Methods Use Message Passing

```
PROXY (calls method)                      SKELETON (handles method)
════════════════════                      ═══════════════════════

1. Write args to SHM
2. Send REQUEST:
   "method call, args in pos 0"
         │                                      │
         │   ┌─────────────────────┐           │
         └──►│ Message Passing:    │           │
             │ REQUEST "call m1"   │──────────►│
             └─────────────────────┘           │
                                               ▼
                                    3. SentWithReplyCallback fires
                                    4. Read args from SHM
                                    5. Execute handler
                                    6. Write return to SHM
                                    7. REPLY "done"
             ┌─────────────────────┐           │
         ◄───│ Message Passing:    │◄──────────┘
             │ REPLY "result ready"│
             └─────────────────────┘
         │
         ▼
8. Proxy reads return value
   from SHM (zero-copy)
```

---

## Configuration

```cpp
struct ServiceProtocolConfig {
    std::string_view identifier;     // Server name (e.g., "lola_service_42_1")
    std::uint32_t max_send_size;     // Max payload for SEND/REQUEST
    std::uint32_t max_reply_size;    // Max payload for REPLY
    std::uint32_t max_notify_size;   // Max payload for NOTIFY
};
```

Messages are **small** — they contain metadata (slot numbers, method IDs), not data. Typical sizes: 8-64 bytes.

> **Related file**: [score/message_passing/service_protocol_config.h](../../score/message_passing/service_protocol_config.h)

---

## Allocation-Free Design (ASIL-B Safe)

The client pre-allocates a pool of send commands at construction:

```
┌──────────────────────────────────────────────────────────┐
│  ClientConnection (pre-allocated at startup)              │
│                                                          │
│  send_pool_: [cmd][cmd][cmd][cmd][cmd]  ← available      │
│  send_queue_: []                        ← pending sends  │
│                                                          │
│  TryQueueMessage(msg):                                   │
│    1. Take cmd from pool (O(1), no malloc)              │
│    2. Copy message bytes into cmd buffer                │
│    3. Push cmd to queue                                 │
│    4. Background thread picks up and sends              │
│                                                          │
│  If pool empty → return ENOBUFS (backpressure)          │
│                                                          │
│  NO dynamic allocation after construction!               │
│  Safe for ASIL-B applications.                          │
└──────────────────────────────────────────────────────────┘
```

---

## Client Identity & Security

The server knows who each client is:

```cpp
struct ClientIdentity {
    pid_t pid;   // Process ID
    uid_t uid;   // User ID  
    gid_t gid;   // Group ID
};

// Linux: obtained from SO_PEERCRED on Unix socket
// QNX: obtained from resource manager connection info
```

This enables:
- **Crash detection**: If a client PID no longer exists → clean up resources
- **Access control**: Verify client has permission to access service
- **Transaction logs**: Track which process wrote what

---

## Threading Model

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  CLIENT PROCESS                         SERVER PROCESS                  │
│                                                                        │
│  ┌──────────────────┐                  ┌──────────────────┐           │
│  │ User Thread      │                  │ User Thread      │           │
│  │ (calls Send())   │                  │ (calls Notify()) │           │
│  └────────┬─────────┘                  └────────┬─────────┘           │
│           │                                      │                     │
│  ┌────────▼─────────┐                  ┌────────▼─────────┐           │
│  │ Library Thread   │                  │ Library Thread   │           │
│  │ (poll/dispatch)  │                  │ (poll/dispatch)  │           │
│  │                  │                  │                  │           │
│  │ • Sends queued   │     SOCKET/      │ • Accepts conns  │           │
│  │   messages       │◄────MESSAGE─────►│ • Dispatches     │           │
│  │ • Receives       │                  │   callbacks      │           │
│  │   NOTIFY/REPLY   │                  │ • Sends NOTIFY   │           │
│  └──────────────────┘                  └──────────────────┘           │
│                                                                        │
│  GUARANTEE: All server callbacks execute sequentially on library thread│
│  GUARANTEE: Same-client messages delivered in order                    │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| [client_connection.h](../../score/message_passing/client_connection.h) | Client implementation |
| [i_client_connection.h](../../score/message_passing/i_client_connection.h) | Client interface |
| [i_server.h](../../score/message_passing/i_server.h) | Server interface |
| [i_server_connection.h](../../score/message_passing/i_server_connection.h) | Per-client session interface |
| [client_server_communication.h](../../score/message_passing/client_server_communication.h) | Protocol message types |
| [service_protocol_config.h](../../score/message_passing/service_protocol_config.h) | Configuration |
| [engine.h](../../score/message_passing/engine.h) | Compile-time transport selection |
| [unix_domain/](../../score/message_passing/unix_domain/) | Linux (Unix socket) implementation |
| [qnx_dispatch/](../../score/message_passing/qnx_dispatch/) | QNX implementation |
| [server_factory.h](../../score/message_passing/server_factory.h) | Factory for creating servers |
| [client_factory.h](../../score/message_passing/client_factory.h) | Factory for creating clients |
| [dependability/](../../score/message_passing/dependability/) | Architecture & safety docs |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **Message passing** | Notification-only IPC (NOT data transfer) |
| **N-to-1** | Multiple clients connect to one server |
| **SEND** | Client → Server, fire-and-forget |
| **REQUEST/REPLY** | Client → Server → Client, synchronous RPC |
| **NOTIFY** | Server → Client, async push notification |
| **Unix Domain** | Linux impl using SOCK_STREAM + poll() |
| **QNX Dispatch** | QNX impl using native message passing |
| **Allocation-free** | Pre-allocated command pool (ASIL-B safe) |
| **Small messages** | Only metadata (slot IDs, method IDs) — not data |

---

## Next: [Part 7 — Safety & Transactions →](part7_safety_transactions.md)

How does LoLa handle crashes? What are transaction logs? How is ASIL-B compliance achieved?

---

## Previous: [← Part 5 — Methods (RPC)](part5_methods_rpc.md)
