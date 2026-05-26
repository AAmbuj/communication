# Part 1: Fundamentals — What LoLa Is and Why It Exists

## The Problem

In a modern car, a single ECU (Electronic Control Unit) runs **dozens of independent applications** as separate processes:

```
┌─────────────────────── Single ECU ───────────────────────┐
│                                                           │
│  ┌─────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │
│  │ Camera  │  │  Sensor  │  │Dashboard │  │    AI    │ │
│  │  App    │  │  Fusion  │  │ Display  │  │ Module   │ │
│  └────┬────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘ │
│       │             │             │              │        │
│       └─────────────┴─────────────┴──────────────┘        │
│                         HOW?                              │
│              They need to share data FAST                 │
│              (< 100 microseconds)                        │
│              and SAFELY (ASIL-B)                         │
└───────────────────────────────────────────────────────────┘
```

### Why Traditional IPC Fails Here

| Method | Problem |
|--------|---------|
| **Pipes** | Data copied: App → Kernel buffer → App (2 copies!) |
| **Sockets** | Same copying + protocol overhead |
| **Message Queues** | Fixed kernel buffer size; newest data dropped when full |
| **All of the above** | Hard to safety-qualify (too much kernel code involved) |

### LoLa's Solution

```
┌──────────┐                              ┌──────────┐
│ Sender   │──── writes directly ────────►│  Shared  │◄──── reads directly ────│ Receiver │
│ Process  │     (no copy)                │  Memory  │      (no copy)          │ Process  │
└──────────┘                              └──────────┘                         └──────────┘

                    ZERO COPIES. NO KERNEL INVOLVEMENT FOR DATA.
                    Only user-space atomics for synchronization.
```

---

## The AUTOSAR `ara::com` Model

LoLa follows the **Adaptive AUTOSAR** communication standard. The key abstraction is the **Service-Oriented Architecture**:

### Core Concepts

```
┌────────────────────────────────────────────────────────────────┐
│                        SERVICE                                  │
│                                                                │
│  A service is a "contract" that defines:                       │
│                                                                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                │
│  │  Events  │    │  Fields  │    │ Methods  │                │
│  │          │    │          │    │          │                │
│  │ One-way  │    │ Stateful │    │ Request/ │                │
│  │ pub/sub  │    │ value +  │    │ Response │                │
│  │ data     │    │ notify   │    │ (RPC)    │                │
│  └──────────┘    └──────────┘    └──────────┘                │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

### Skeleton vs Proxy (The Restaurant Analogy)

```
    ╔══════════════════╗                    ╔══════════════════╗
    ║    SKELETON      ║                    ║      PROXY       ║
    ║   (The Kitchen)  ║                    ║  (The Customer)  ║
    ╠══════════════════╣                    ╠══════════════════╣
    ║                  ║                    ║                  ║
    ║ • Produces data  ║                    ║ • Consumes data  ║
    ║ • Offers service ║   ◄── Service ──► ║ • Finds service  ║
    ║ • Sends events   ║     Discovery     ║ • Subscribes     ║
    ║ • Handles methods║                    ║ • Calls methods  ║
    ║                  ║                    ║                  ║
    ╚══════════════════╝                    ╚══════════════════╝
    
    One Skeleton can serve MANY Proxies (1:N relationship)
```

---

## The Complete Usage Flow (With Code)

### Step 1: Initialize the Runtime

```cpp
#include "score/mw/com/runtime.h"

int main(int argc, char** argv) {
    // Initialize mw::com with configuration
    score::mw::com::runtime::Initialize(argc, argv);
    // ... use LoLa ...
}
```

> **Related file**: [score/mw/com/runtime.h](../../score/mw/com/runtime.h)

### Step 2: Skeleton (Publisher) — Offer and Send

```cpp
#include "score/mw/com/types.h"

// 1. Create skeleton from instance specifier
auto instance_spec = InstanceSpecifier{"my_app/sensor_service"};
auto skeleton_result = MySkeleton::Create(instance_spec);
auto& skeleton = skeleton_result.value();

// 2. Make service discoverable
skeleton.OfferService();

// 3. Allocate a slot in shared memory
auto sample = skeleton.temperature_event_.Allocate();

// 4. Write data directly into shared memory (zero-copy!)
sample->temperature = 42.5f;
sample->timestamp = now();

// 5. Publish — notifies all subscribers
skeleton.temperature_event_.Send(std::move(sample));

// 6. When shutting down
skeleton.StopOfferService();
```

> **Related files**: 
> - [score/mw/com/impl/skeleton_base.h](../../score/mw/com/impl/skeleton_base.h)
> - [score/mw/com/impl/skeleton_event.h](../../score/mw/com/impl/skeleton_event.h)

### Step 3: Proxy (Subscriber) — Find, Subscribe, Receive

```cpp
// 1. Find the service
auto handles = MyProxy::FindService(instance_spec);
auto& handle = handles.value()[0];

// 2. Create proxy from discovered handle
auto proxy_result = MyProxy::Create(std::move(handle));
auto& proxy = proxy_result.value();

// 3. Subscribe to event (buffer up to 10 samples)
proxy.temperature_event_.Subscribe(10);

// 4. Receive data — pointer goes directly to shared memory!
proxy.temperature_event_.GetNewSamples(
    [](SamplePtr<TemperatureData> sample) {
        // 'sample' is a smart pointer to shared memory
        // NO COPY happened — you're reading the producer's memory directly
        std::cout << "Temperature: " << sample->temperature << std::endl;
    },
    10  // max samples to read
);

// 5. Cleanup
proxy.temperature_event_.Unsubscribe();
```

> **Related files**:
> - [score/mw/com/impl/proxy_base.h](../../score/mw/com/impl/proxy_base.h)
> - [score/mw/com/impl/proxy_event.h](../../score/mw/com/impl/proxy_event.h)

---

## Key Types (Your API Vocabulary)

| Type | What It Is | Analogy |
|------|-----------|---------|
| `InstanceSpecifier` | Design-time port name (e.g., "my_app/sensor") | Restaurant's address |
| `InstanceIdentifier` | Runtime-resolved unique ID | Table reservation number |
| `HandleType` | A discovered service you can connect to | The waiter saying "your table is ready" |
| `SamplePtr<T>` | Smart pointer to received data in SHM | The plate of food served to you |
| `SampleAllocateePtr<T>` | Smart pointer to allocated write slot | Empty plate the chef prepares on |
| `SubscriptionState` | Subscribed / NotSubscribed / Pending | Your order status |
| `EventReceiveHandler` | Callback when new data arrives | Bell that rings when food is ready |
| `FindServiceHandler<T>` | Callback when service appears/disappears | Notification that restaurant opened |

> **Related file**: [score/mw/com/types.h](../../score/mw/com/types.h)

---

## The Layered Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ Layer 4: YOUR APPLICATION                                                    │
│                                                                             │
│   Uses: InstanceSpecifier, Skeleton::Create(), Proxy::FindService(),        │
│          Subscribe(), GetNewSamples(), Allocate(), Send()                   │
│                                                                             │
│   Files: score/mw/com/example/ipc_bridge/                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ Layer 3: BINDING-INDEPENDENT API  (score/mw/com/impl/)                      │
│                                                                             │
│   • SkeletonBase / ProxyBase       — lifecycle management                   │
│   • SkeletonEvent / ProxyEvent     — typed event pub/sub                    │
│   • ServiceDiscovery               — abstract service matching              │
│   • Runtime                        — initialization & config                │
│                                                                             │
│   This layer does NOT know about shared memory!                             │
│   It delegates to the "binding" below via interfaces.                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ Layer 2: LOLA BINDING  (score/mw/com/impl/bindings/lola/)                   │
│                                                                             │
│   • Shared memory allocation & management                                   │
│   • Lock-free slot-based data exchange                                      │
│   • Subscription state machines                                             │
│   • Transaction logs for crash safety                                       │
│   • Service discovery via flag files                                        │
│                                                                             │
│   THIS IS WHERE THE MAGIC HAPPENS — zero-copy, no-kernel IPC               │
├─────────────────────────────────────────────────────────────────────────────┤
│ Layer 1: MESSAGE PASSING  (score/message_passing/)                           │
│                                                                             │
│   • Used ONLY for notifications ("new data available")                      │
│   • NOT used for actual data transfer                                       │
│   • Linux: Unix domain sockets                                              │
│   • QNX: QNX dispatch/pulse mechanism                                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ Layer 0: OPERATING SYSTEM                                                    │
│                                                                             │
│   • mmap() / shm_open() — creates shared memory regions                    │
│   • File system — flag files for service discovery                          │
│   • POSIX / QNX APIs                                                        │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Real Example: IPC Bridge Application

The project includes a complete working example at:
**[score/mw/com/example/ipc_bridge/](../../score/mw/com/example/ipc_bridge/)**

### Files in the Example

| File | Purpose |
|------|---------|
| [main.cpp](../../score/mw/com/example/ipc_bridge/main.cpp) | Entry point, parses CLI args, decides skeleton or proxy mode |
| [sample_sender_receiver.h](../../score/mw/com/example/ipc_bridge/sample_sender_receiver.h) | Class with `RunAsSkeleton()` and `RunAsProxy()` methods |
| [sample_sender_receiver.cpp](../../score/mw/com/example/ipc_bridge/sample_sender_receiver.cpp) | Full implementation of send/receive logic |
| [datatype.h](../../score/mw/com/example/ipc_bridge/datatype.h) | Data structures exchanged (MapApiLanesStamped) |
| [ipc_bridge.rs](../../score/mw/com/example/ipc_bridge/ipc_bridge.rs) | Rust version of the same example |
| [etc/](../../score/mw/com/example/ipc_bridge/etc/) | Configuration JSON files |

### Running the Example

```bash
# Terminal 1: Start as skeleton (sender)
bazel run //score/mw/com/example/ipc_bridge -- --mode skeleton --cycle-time 100 --num-cycles 50

# Terminal 2: Start as proxy (receiver)
bazel run //score/mw/com/example/ipc_bridge -- --mode recv --cycle-time 100 --num-cycles 50
```

---

## Summary

| Concept | One-line explanation |
|---------|---------------------|
| **LoLa** | Zero-copy shared-memory IPC for automotive ECUs |
| **Skeleton** | Service provider — offers service, sends data |
| **Proxy** | Service consumer — finds service, receives data |
| **Event** | One-way pub/sub data stream (Skeleton → Proxy) |
| **Field** | Stateful value with get/set/notify |
| **Method** | RPC: Proxy calls → Skeleton executes → returns result |
| **Binding** | The implementation layer (LoLa = shared memory) |
| **Zero-copy** | SamplePtr gives direct pointer to shared memory — no copies |

---

## Next: [Part 2 — Service Discovery →](part2_service_discovery.md)

How does `FindService()` actually find the skeleton? What are flag files? How do identifiers resolve?
