# Part 4: Event System — Publishing and Subscribing to Data

## Overview

Events are the **primary communication pattern** in LoLa. A skeleton publishes events; proxies subscribe and receive them. This part covers how `SkeletonEvent` and `ProxyEvent` wrap the shared memory machinery from Part 3 into a user-friendly API.

---

## Event Architecture — Class Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  SKELETON SIDE (Publisher)              PROXY SIDE (Subscriber)              │
│                                                                             │
│  ┌──────────────────────┐              ┌──────────────────────┐            │
│  │    SkeletonEvent<T>  │              │    ProxyEvent<T>     │            │
│  │    (typed, user API) │              │    (typed, user API) │            │
│  │                      │              │                      │            │
│  │  • Allocate()        │              │  • GetNewSamples()   │            │
│  │  • Send()            │              │  • Subscribe()       │            │
│  └──────────┬───────────┘              └──────────┬───────────┘            │
│             │ inherits                            │ inherits               │
│  ┌──────────▼───────────┐              ┌──────────▼───────────┐            │
│  │  SkeletonEventBase   │              │   ProxyEventBase     │            │
│  │  (type-agnostic)     │              │   (type-agnostic)    │            │
│  │                      │              │                      │            │
│  │  • PrepareOffer()    │              │  • Subscribe(count)  │            │
│  │  • PrepareStopOffer()│              │  • Unsubscribe()     │            │
│  │                      │              │  • SetReceiveHandler │            │
│  │                      │              │  • GetSubscription   │            │
│  │                      │              │    State()           │            │
│  └──────────┬───────────┘              └──────────┬───────────┘            │
│             │ delegates to                        │ delegates to           │
│  ┌──────────▼───────────┐              ┌──────────▼───────────┐            │
│  │  SkeletonEventBinding│              │  ProxyEventBinding   │            │
│  │  (interface)         │              │  (interface)         │            │
│  └──────────┬───────────┘              └──────────┬───────────┘            │
│             │ implemented by                      │ implemented by         │
│  ┌──────────▼───────────┐              ┌──────────▼───────────┐            │
│  │  LoLa SkeletonEvent  │              │  LoLa ProxyEvent     │            │
│  │  (shared memory impl)│              │  (shared memory impl)│            │
│  │  bindings/lola/       │              │  bindings/lola/       │            │
│  └──────────────────────┘              └──────────────────────┘            │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Subscription State Machine

The proxy event goes through a **state machine** that manages its lifecycle:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    SUBSCRIPTION STATE MACHINE                             │
│                                                                         │
│                                                                         │
│  ┌──────────────────┐    Subscribe()    ┌─────────────────────┐        │
│  │                  │──────────────────►│                     │        │
│  │  NOT_SUBSCRIBED  │                   │ SUBSCRIPTION_PENDING│        │
│  │                  │◄──────────────────│                     │        │
│  └──────────────────┘   StopOffer /     └──────────┬──────────┘        │
│         ▲                Unsubscribe               │                   │
│         │                                          │ ReOffer           │
│         │                                          │ (Service found!)  │
│         │                                          ▼                   │
│         │               Unsubscribe     ┌──────────────────────┐       │
│         └───────────────────────────────│                      │       │
│                                         │     SUBSCRIBED       │       │
│         ┌───────────────────────────────│                      │       │
│         │               StopOffer       └──────────────────────┘       │
│         ▼                                                              │
│  ┌──────────────────┐                                                  │
│  │  NOT_SUBSCRIBED  │  (Clean state after stop offer)                  │
│  └──────────────────┘                                                  │
│                                                                         │
│  TRANSITIONS:                                                           │
│  • Subscribe()    → User initiates subscription                        │
│  • ReOffer        → Service discovery: skeleton became available        │
│  • Unsubscribe()  → User cancels subscription                          │
│  • StopOffer      → Service discovery: skeleton disappeared            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### States Explained

| State | Meaning | What can the proxy do? |
|-------|---------|----------------------|
| `kNotSubscribed` | Not connected to any event data | Call `Subscribe()` |
| `kSubscriptionPending` | Waiting for skeleton to appear/confirm | Wait (or `Unsubscribe()` to cancel) |
| `kSubscribed` | Actively receiving data | `GetNewSamples()`, `SetReceiveHandler()` |

> **Related file**: [score/mw/com/impl/subscription_state.h](../../score/mw/com/impl/subscription_state.h)
> **Related file**: [score/mw/com/impl/bindings/lola/subscription_state_machine.h](../../score/mw/com/impl/bindings/lola/subscription_state_machine.h)

---

## Skeleton Event — Publishing Data

### API Flow

```cpp
// In your generated skeleton class:
class MySkeleton : public SkeletonBase {
  public:
    SkeletonEvent<TemperatureData> temperature_event_;
};

// Usage:
auto& skeleton = my_skeleton;

// Step 1: Allocate a slot (returns pointer to shared memory)
auto sample_result = skeleton.temperature_event_.Allocate();
auto sample = std::move(sample_result.value());

// Step 2: Write data (directly in shared memory!)
sample->temperature = 42.5f;
sample->unit = "celsius";

// Step 3: Send (marks slot ready, notifies subscribers)
auto send_result = skeleton.temperature_event_.Send(std::move(sample));
```

### Internal Flow Diagram

```
skeleton.temperature_event_.Allocate()
         │
         ▼
┌─────────────────────────────────────────┐
│ SkeletonEvent<T>::Allocate()            │
│                                         │
│  Delegates to binding:                  │
│  lola::SkeletonEvent::Allocate()        │
│         │                               │
│         ▼                               │
│  ┌─────────────────────────────┐        │
│  │ Find free slot (lock-free)  │        │
│  │ • Scan all slots            │        │
│  │ • Pick: lowest timestamp    │        │
│  │         AND readers == 0    │        │
│  │ • CAS: mark IN_WRITING     │        │
│  └──────────────┬──────────────┘        │
│                 │                        │
│                 ▼                        │
│  Return SampleAllocateePtr<T>           │
│  (points to slot in shared memory)      │
└─────────────────────────────────────────┘

skeleton.temperature_event_.Send(sample)
         │
         ▼
┌─────────────────────────────────────────┐
│ SkeletonEvent<T>::Send()                │
│                                         │
│  1. CAS: set timestamp on slot          │
│     (marks READY for consumers)         │
│                                         │
│  2. Trigger notification via            │
│     message passing channel             │
│     ("new data in slot N!")             │
└─────────────────────────────────────────┘
```

> **Related files**:
> - [score/mw/com/impl/skeleton_event_base.h](../../score/mw/com/impl/skeleton_event_base.h)
> - [score/mw/com/impl/skeleton_event.h](../../score/mw/com/impl/skeleton_event.h)
> - [score/mw/com/impl/bindings/lola/skeleton_event.h](../../score/mw/com/impl/bindings/lola/skeleton_event.h)

---

## Proxy Event — Receiving Data

### Two Reception Modes

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  MODE 1: POLLING                       MODE 2: ASYNC CALLBACK          │
│  (You ask for data)                    (You get notified)              │
│                                                                        │
│  while (running) {                     proxy.event_.SetReceiveHandler( │
│    proxy.event_.GetNewSamples(           []() {                        │
│      [](SamplePtr<T> s) {                  // Called when data arrives │
│        process(*s);                        proxy.event_.GetNewSamples( │
│      },                                      [](SamplePtr<T> s) {     │
│      max_count                                 process(*s);            │
│    );                                        }, max);                  │
│    sleep(10ms);                          }                             │
│  }                                     );                              │
│                                                                        │
│  Pro: Simple, predictable timing       Pro: Low latency, immediate     │
│  Con: Wastes CPU if no data            Con: Called from worker thread  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Subscribe() Flow

```cpp
// Subscribe with buffer for max 10 concurrent samples
auto result = proxy.temperature_event_.Subscribe(10);
```

```
Subscribe(max_sample_count=10)
         │
         ▼
┌─────────────────────────────────────────┐
│ ProxyEventBase::Subscribe()             │
│                                         │
│  1. Validate: not already subscribed    │
│                                         │
│  2. Set max_sample_count = 10           │
│     (SampleReferenceTracker limit)      │
│                                         │
│  3. Delegate to binding:                │
│     SubscriptionStateMachine            │
│       .SubscribeEvent(10)               │
│                                         │
│  4. State: NOT_SUBSCRIBED               │
│         → SUBSCRIPTION_PENDING          │
│                                         │
│  5. If skeleton already available:      │
│     → immediately → SUBSCRIBED          │
│     Create SlotCollector for reading    │
└─────────────────────────────────────────┘
```

### GetNewSamples() Flow

```cpp
proxy.temperature_event_.GetNewSamples(
    [](SamplePtr<TemperatureData> sample) {
        // Each invocation gives you ONE sample
        std::cout << sample->temperature << std::endl;
        // sample is a direct pointer into shared memory (zero-copy!)
    },
    5  // max 5 samples per call
);
```

```
GetNewSamples(callback, max=5)
         │
         ▼
┌─────────────────────────────────────────────────────────────────────┐
│ ProxyEvent<T>::GetNewSamples()                                       │
│                                                                     │
│  1. Check: subscription state == SUBSCRIBED                         │
│                                                                     │
│  2. SlotCollector scans all slots:                                  │
│     ┌──────────────────────────────────────────────────────┐       │
│     │ For each slot:                                        │       │
│     │   if slot.timestamp > my_last_seen_timestamp:        │       │
│     │     CAS: increment subscriber_count                   │       │
│     │     Add to "new samples" list                         │       │
│     └──────────────────────────────────────────────────────┘       │
│                                                                     │
│  3. Sort new samples by timestamp (oldest first)                    │
│                                                                     │
│  4. For each new sample (up to max=5):                             │
│     ┌──────────────────────────────────────────────────────┐       │
│     │ Create SamplePtr<T> pointing to slot data            │       │
│     │ Invoke callback(sample_ptr)                           │       │
│     │                                                       │       │
│     │ SampleReferenceTracker: count++                       │       │
│     │ (ensures we don't exceed max_sample_count=10)         │       │
│     └──────────────────────────────────────────────────────┘       │
│                                                                     │
│  5. Return: number of samples delivered                             │
│                                                                     │
│  When SamplePtr<T> goes out of scope:                              │
│    → SlotDecrementer: CAS decrement subscriber_count               │
│    → SampleReferenceTracker: count--                               │
│    → Slot becomes available for skeleton to reuse                  │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## max_sample_count — Resource Management

The `max_sample_count` parameter in `Subscribe()` limits how many `SamplePtr`s a proxy can hold simultaneously:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Subscribe(max_sample_count = 3)                                │
│                                                                 │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐                         │
│  │SamplePtr│ │SamplePtr│ │SamplePtr│  ← Holding 3 samples     │
│  │ (slot 0)│ │ (slot 1)│ │ (slot 4)│                          │
│  └─────────┘ └─────────┘ └─────────┘                         │
│                                                                 │
│  GetFreeSampleCount() → 0   (all used!)                        │
│  GetNewSamples() → would return 0 new samples                  │
│                                                                 │
│  // Release one sample:                                         │
│  sample_ptr_0.reset();  // or let it go out of scope            │
│                                                                 │
│  GetFreeSampleCount() → 1   (one slot freed!)                  │
│  GetNewSamples() → can now deliver 1 sample                    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

FORMULA for total slots needed:
  num_storage_slots = 1 + Σ(max_sample_count of all subscribers)
  
  Example: 3 subscribers with max=5 each → 1 + 15 = 16 slots
  (The +1 is for the skeleton's current write slot)
```

> **Related file**: [score/mw/com/impl/sample_reference_tracker.h](../../score/mw/com/impl/sample_reference_tracker.h)

---

## EventReceiveHandler — Async Notifications

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    RECEIVE HANDLER WRAPPING CHAIN                         │
│                                                                         │
│  User provides:                                                         │
│  ┌──────────────────────────┐                                          │
│  │ callback<void()>         │  ← Your function                         │
│  └────────────┬─────────────┘                                          │
│               │ wrapped in                                              │
│  ┌────────────▼─────────────┐                                          │
│  │ Context Tracking         │  ← Sets is_in_receive_handler flag       │
│  │ (prevents deadlock)      │    (so Unsubscribe inside handler works) │
│  └────────────┬─────────────┘                                          │
│               │ wrapped in                                              │
│  ┌────────────▼─────────────┐                                          │
│  │ ScopedEventReceiveHandler│  ← RAII: handler can't be called after   │
│  │ (scope guard)            │    scope expires (safe destruction)       │
│  └────────────┬─────────────┘                                          │
│               │ stored as                                               │
│  ┌────────────▼─────────────┐                                          │
│  │ shared_ptr → weak_ptr    │  ← ProxyEventBase owns shared_ptr       │
│  │                          │    Binding gets weak_ptr (safe callback) │
│  └──────────────────────────┘                                          │
│                                                                         │
│  When new data arrives:                                                 │
│  1. Message passing notification received                              │
│  2. Binding locks weak_ptr → shared_ptr                                │
│  3. If lock succeeds: invoke handler                                   │
│  4. If lock fails: handler was unset, skip                            │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Usage:

```cpp
// Set up async notification
proxy.temperature_event_.SetReceiveHandler(
    []() {
        // Called from worker thread when new data arrives!
        // Usually you'd call GetNewSamples() here:
        proxy.temperature_event_.GetNewSamples(
            [](SamplePtr<TemperatureData> s) {
                process(*s);
            }, max);
    }
);

// Later: safely remove handler (blocks until any running handler completes)
proxy.temperature_event_.UnsetReceiveHandler();
```

> **Related files**:
> - [score/mw/com/impl/event_receive_handler.h](../../score/mw/com/impl/event_receive_handler.h)
> - [score/mw/com/impl/scoped_event_receive_handler.h](../../score/mw/com/impl/scoped_event_receive_handler.h)

---

## Fields — Events with State

A **Field** is an Event with additional capabilities:

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  EVENT                              FIELD                          │
│  ─────                              ─────                          │
│  • One-way data stream              • Has a "current value"        │
│  • No initial value                 • Initial value on subscribe   │
│  • Subscribe + GetNewSamples        • Get() to read current        │
│  • Fire-and-forget                  • Set() to update (optional)   │
│                                     • Notifies on change           │
│                                                                    │
│  Use for:                           Use for:                       │
│  • Sensor readings (temp/speed)     • Vehicle state (gear/mode)    │
│  • Camera frames                    • Configuration values         │
│  • Log messages                     • Status indicators            │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

```
┌──────────────────────────────────────────────────────────────────┐
│  FIELD CLASS HIERARCHY                                            │
│                                                                  │
│  ┌────────────────────┐          ┌───────────────────┐          │
│  │ SkeletonField<T>   │          │ ProxyField<T>     │          │
│  │                    │          │                   │          │
│  │ • Update(value)    │          │ • Get()           │          │
│  │   (sends to all    │          │   (read current)  │          │
│  │    subscribers)    │          │ • Subscribe()     │          │
│  │                    │          │   (get notified   │          │
│  └────────┬───────────┘          │    on change)     │          │
│           │ inherits             └───────┬───────────┘          │
│  ┌────────▼───────────┐          ┌───────▼───────────┐          │
│  │ SkeletonFieldBase  │          │ ProxyFieldBase    │          │
│  │                    │          │                   │          │
│  │ Inherits from:     │          │ Inherits from:    │          │
│  │ SkeletonEventBase  │          │ ProxyEventBase    │          │
│  └────────────────────┘          └───────────────────┘          │
│                                                                  │
│  Fields REUSE the event infrastructure underneath!               │
│  They just add Get/Set semantics on top.                        │
└──────────────────────────────────────────────────────────────────┘
```

> **Related files**:
> - [score/mw/com/impl/skeleton_field_base.h](../../score/mw/com/impl/skeleton_field_base.h)
> - [score/mw/com/impl/proxy_field_base.h](../../score/mw/com/impl/proxy_field_base.h)

---

## Mixed-Criticality (ASIL-B + QM)

Events support simultaneous **ASIL-B** (safety-critical) and **QM** (non-safety) subscribers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  SKELETON (offers at ASIL-B)                                        │
│       │                                                             │
│       ▼                                                             │
│  ┌──────────────────────────────────────────────┐                  │
│  │  SHARED MEMORY                                │                  │
│  │                                              │                  │
│  │  ┌─────────────────────┐  ┌──────────────┐  │                  │
│  │  │ ASIL-B Control SHM  │  │ QM Control   │  │                  │
│  │  │ (safety-qualified)  │  │ SHM          │  │                  │
│  │  └─────────────────────┘  └──────────────┘  │                  │
│  │                                              │                  │
│  │  ┌───────────────────────────────────────┐   │                  │
│  │  │ DATA SHM (shared by both)             │   │                  │
│  │  └───────────────────────────────────────┘   │                  │
│  └──────────────────────────────────────────────┘                  │
│       │                              │                              │
│       ▼                              ▼                              │
│  ASIL-B Proxy                   QM Proxy                           │
│  (braking system)               (dashboard display)                │
│                                                                     │
│  RULE: QM corruption cannot affect ASIL-B operation                │
│  HOW: Separate control structures, bounded retries for QM slots    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Complete Lifecycle Example

```cpp
// ═══════ SKELETON SIDE ═══════
auto skeleton = MySkeleton::Create(specifier).value();
skeleton.OfferService();

// Periodically send data:
while (running) {
    auto sample = skeleton.temperature_.Allocate().value();
    sample->value = read_sensor();
    skeleton.temperature_.Send(std::move(sample));
    sleep(100ms);
}

skeleton.StopOfferService();

// ═══════ PROXY SIDE ═══════
auto handle = MyProxy::FindService(specifier).value()[0];
auto proxy = MyProxy::Create(std::move(handle)).value();

// Subscribe (max 5 samples buffered)
proxy.temperature_.Subscribe(5);

// Option A: Polling
while (running) {
    proxy.temperature_.GetNewSamples(
        [](SamplePtr<TemperatureData> s) {
            display(s->value);
        }, 5);
    sleep(50ms);
}

// Option B: Async handler
proxy.temperature_.SetReceiveHandler([&]() {
    proxy.temperature_.GetNewSamples(
        [](SamplePtr<TemperatureData> s) { display(s->value); }, 5);
});

// Cleanup
proxy.temperature_.UnsetReceiveHandler();
proxy.temperature_.Unsubscribe();
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| [skeleton_event_base.h](../../score/mw/com/impl/skeleton_event_base.h) | Base class for skeleton events |
| [skeleton_event.h](../../score/mw/com/impl/skeleton_event.h) | Typed skeleton event template |
| [proxy_event_base.h](../../score/mw/com/impl/proxy_event_base.h) | Base class for proxy events |
| [proxy_event.h](../../score/mw/com/impl/proxy_event.h) | Typed proxy event template |
| [subscription_state.h](../../score/mw/com/impl/subscription_state.h) | Subscription state enum |
| [subscription_state_machine.h](../../score/mw/com/impl/bindings/lola/subscription_state_machine.h) | State machine implementation |
| [event_receive_handler.h](../../score/mw/com/impl/event_receive_handler.h) | Callback type definition |
| [scoped_event_receive_handler.h](../../score/mw/com/impl/scoped_event_receive_handler.h) | RAII handler wrapper |
| [sample_reference_tracker.h](../../score/mw/com/impl/sample_reference_tracker.h) | max_sample_count enforcement |
| [slot_collector.h](../../score/mw/com/impl/bindings/lola/slot_collector.h) | Finds new samples in slots |
| [skeleton_field_base.h](../../score/mw/com/impl/skeleton_field_base.h) | Field on skeleton side |
| [proxy_field_base.h](../../score/mw/com/impl/proxy_field_base.h) | Field on proxy side |
| [score/mw/com/design/events_fields/](../../score/mw/com/design/events_fields/) | Design documentation |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **SkeletonEvent** | Allocate slot → write data → Send (mark ready + notify) |
| **ProxyEvent** | Subscribe → GetNewSamples (scan slots, return SamplePtrs) |
| **Subscribe(N)** | Limits proxy to hold max N SamplePtrs concurrently |
| **State Machine** | NotSubscribed → Pending → Subscribed (lifecycle) |
| **SetReceiveHandler** | Async callback when new data arrives |
| **Field** | Event + current value (Get/Set/Notify pattern) |
| **Mixed-criticality** | Separate ASIL/QM control; shared data |

---

## Next: [Part 5 — Methods (RPC) →](part5_methods_rpc.md)

How does request/response work? How are method calls routed through shared memory?

---

## Previous: [← Part 3 — Shared Memory & Zero-Copy](part3_shared_memory.md)
