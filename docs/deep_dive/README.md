# LoLa Deep Dive - Learning Guide

A structured, step-by-step guide to fully understand the LoLa (Low Latency) Communication Module.

## Learning Path

| Part | Topic | Description |
|------|-------|-------------|
| [Part 1](part1_fundamentals.md) | **Fundamentals** | What LoLa solves, AUTOSAR concepts, Skeleton/Proxy pattern |
| [Part 2](part2_service_discovery.md) | **Service Discovery** | How services find each other (flag files, identifiers) |
| [Part 3](part3_shared_memory.md) | **Shared Memory & Zero-Copy** | The LoLa binding, memory pools, lock-free design |
| [Part 4](part4_event_system.md) | **Event System** | Publishing and subscribing to data |
| [Part 5](part5_methods_rpc.md) | **Methods (RPC)** | Request/Response pattern |
| [Part 6](part6_message_passing.md) | **Message Passing Layer** | Low-level OS communication (notifications) |
| [Part 7](part7_safety_transactions.md) | **Safety & Transactions** | Transaction logs, rollback, ASIL-B |
| [Part 8](part8_configuration.md) | **Configuration & Deployment** | JSON manifests, runtime config |
| [Part 9](part9_building_testing.md) | **Building & Testing** | Bazel targets, test patterns |
| [Part 10](part10_contributing.md) | **Contributing** | Where to start, workflow |

## Architecture Overview (Full System)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         YOUR APPLICATION                                     │
│                                                                             │
│   ┌──────────────────┐                    ┌──────────────────┐              │
│   │  Skeleton (Pub)  │                    │   Proxy (Sub)    │              │
│   │  - OfferService  │                    │  - FindService   │              │
│   │  - Send()        │                    │  - Subscribe     │              │
│   │  - Allocate()    │                    │  - GetNewSamples │              │
│   └────────┬─────────┘                    └────────┬─────────┘              │
│            │                                       │                        │
└────────────┼───────────────────────────────────────┼────────────────────────┘
             │                                       │
┌────────────┼───────────────────────────────────────┼────────────────────────┐
│            │     score/mw/com/impl/ (API Layer)    │                        │
│   ┌────────▼─────────┐                    ┌───────▼──────────┐             │
│   │  SkeletonBase    │                    │   ProxyBase      │             │
│   │  SkeletonEvent   │◄───────────────────│   ProxyEvent     │             │
│   │  SkeletonField   │  Service Discovery │   ProxyField     │             │
│   │  SkeletonMethod  │                    │   ProxyMethod    │             │
│   └────────┬─────────┘                    └───────┬──────────┘             │
│            │                                       │                        │
└────────────┼───────────────────────────────────────┼────────────────────────┘
             │                                       │
┌────────────┼───────────────────────────────────────┼────────────────────────┐
│            │   score/mw/com/impl/bindings/lola/    │                        │
│   ┌────────▼─────────┐                    ┌───────▼──────────┐             │
│   │  LoLa Skeleton   │                    │  LoLa Proxy      │             │
│   │                  │    SHARED MEMORY   │                  │             │
│   │  Writes data ────┼──►┌───────────┐───►┼── Reads data     │             │
│   │  to SHM slot     │   │ Memory    │    │  from SHM slot   │             │
│   │                  │   │ Pool      │    │                  │             │
│   └────────┬─────────┘   └───────────┘    └───────┬──────────┘             │
│            │                                       │                        │
│            │         NOTIFICATION ONLY             │                        │
│            └───────────────────┬───────────────────┘                        │
│                                │                                            │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────────┐
│                                │                                            │
│         score/message_passing/ (Notification Channel)                       │
│                                │                                            │
│   ┌────────────────────────────▼───────────────────────────────┐            │
│   │  Unix Domain Sockets (Linux) │ QNX Dispatch (QNX)          │            │
│   │  "Hey, new data available!"  │ "Hey, new data available!"  │            │
│   └────────────────────────────────────────────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                 │
┌────────────────────────────────┼────────────────────────────────────────────┐
│              Operating System (Linux / QNX Neutrino)                         │
│                                                                             │
│   mmap() / shm_open() / POSIX shared memory / QNX resource manager          │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagram

```
    SKELETON (Publisher)                              PROXY (Subscriber)
    ═══════════════════                              ═════════════════

    1. Create Skeleton                               
           │                                         
    2. OfferService()                                
           │                 ┌──────────────┐        
           ├────────────────►│   Service    │        
           │   "I'm here!"  │  Discovery   │        
           │                 │  (Flag File) │        3. FindService()
           │                 └──────┬───────┘◄───────────────┤
           │                        │                        │
           │                        │ "Match found!"         │
           │                        ├───────────────────────►│
           │                                                 │
           │                                          4. Subscribe()
           │                                                 │
           │         ┌──────────────────────┐                │
    5. Allocate()    │                      │                │
           │         │   SHARED MEMORY      │                │
           ├────────►│   ┌──┬──┬──┬──┐     │                │
    6. Write data    │   │S1│S2│S3│S4│ ... │                │
           │         │   └──┴──┴──┴──┘     │                │
    7. Send()        │          │           │         8. GetNewSamples()
           │         │          │           │                │
           │         │          └───────────┼───────────────►│
           │         │    (Zero-Copy read)  │                │
           │         └──────────────────────┘         9. Process data
           │                                                 │
           │◄─── Notification (msg passing) ────────────────►│
           │      "new data in slot N"                       │
```

## Quick Reference - Key Source Files

| Concept | File |
|---------|------|
| All public types | [score/mw/com/types.h](../../score/mw/com/types.h) |
| Runtime init | [score/mw/com/runtime.h](../../score/mw/com/runtime.h) |
| Skeleton base class | [score/mw/com/impl/skeleton_base.h](../../score/mw/com/impl/skeleton_base.h) |
| Proxy base class | [score/mw/com/impl/proxy_base.h](../../score/mw/com/impl/proxy_base.h) |
| LoLa binding (core) | [score/mw/com/impl/bindings/lola/](../../score/mw/com/impl/bindings/lola/) |
| Message passing | [score/message_passing/](../../score/message_passing/) |
| Full example app | [score/mw/com/example/ipc_bridge/](../../score/mw/com/example/ipc_bridge/) |
| Design docs | [score/mw/com/design/](../../score/mw/com/design/) |
