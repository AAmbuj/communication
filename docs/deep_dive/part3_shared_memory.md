# Part 3: Shared Memory & Zero-Copy — The Heart of LoLa

## Overview

This is where LoLa's magic lives. After Service Discovery connects a skeleton to a proxy (Part 2), they communicate through **shared memory** with **zero copies** of data. This part explains HOW that works internally.

---

## The Two Shared Memory Regions

Every offered service creates **two separate shared memory objects**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    PER-SERVICE INSTANCE                           │
│                                                                 │
│  ┌───────────────────────────┐  ┌───────────────────────────┐  │
│  │    DATA Shared Memory     │  │   CONTROL Shared Memory    │  │
│  │                           │  │                           │  │
│  │  Actual sample values     │  │  Slot status (atomics)    │  │
│  │  (temperature, images,    │  │  Subscription counts      │  │
│  │   sensor readings...)     │  │  Transaction logs         │  │
│  │                           │  │  PID/crash mapping        │  │
│  │  Size: depends on data    │  │  Size: small (metadata)   │  │
│  └───────────────────────────┘  └───────────────────────────┘  │
│                                                                 │
│  WHY SEPARATE?                                                  │
│  • Data SHM: accessed for actual read/write (cache-hot)        │
│  • Control SHM: accessed for synchronization (separate cache)  │
│  • Reduces cache contention between data and control paths     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Slot-Based Architecture

The central concept is **slots** — fixed-size pre-allocated positions in shared memory:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  EVENT: "temperature" (configured with 4 slots)                         │
│                                                                         │
│  DATA Shared Memory:                                                    │
│  ┌────────────┬────────────┬────────────┬────────────┐                 │
│  │  Slot 0    │  Slot 1    │  Slot 2    │  Slot 3    │                 │
│  │            │            │            │            │                 │
│  │ temp=22.5  │ temp=23.1  │ (writing)  │ temp=21.8  │                 │
│  │ ts=1000    │ ts=1200    │            │ ts=800     │                 │
│  └────────────┴────────────┴────────────┴────────────┘                 │
│       ↕              ↕            ↕            ↕                        │
│  CONTROL Shared Memory:                                                 │
│  ┌────────────┬────────────┬────────────┬────────────┐                 │
│  │  Status 0  │  Status 1  │  Status 2  │  Status 3  │                 │
│  │            │            │            │            │                 │
│  │ timestamp:3│ timestamp:4│ IN_WRITING │ timestamp:2│                 │
│  │ readers: 1 │ readers: 0 │ readers: 0 │ readers: 0 │                 │
│  └────────────┴────────────┴────────────┴────────────┘                 │
│                                                                         │
│  Slot 1 has highest timestamp → it's the NEWEST data                   │
│  Slot 0 has 1 reader → a proxy is still holding a SamplePtr to it      │
│  Slot 2 is being written → skeleton allocated it, filling data          │
│  Slot 3 has old data → can be reused for next allocation               │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Why Slots?

- **No dynamic allocation at runtime** — all slots are pre-allocated at `OfferService()` time
- **Lock-free** — each slot has an atomic status, no mutexes needed
- **Bounded memory** — number of slots is configured, preventing memory growth
- **"Last-is-best"** — newest data overwrites oldest slot, no queue overflow issues

---

## The EventSlotStatus — 64-bit Atomic Magic

Each slot's status is packed into a **single 64-bit atomic integer**:

```
┌────────────────────────────────────────────────────────────────────┐
│                    EventSlotStatus (64 bits)                         │
│                                                                    │
│  ┌──────────────────────────┬──────────────────────────┐          │
│  │  Upper 32 bits           │  Lower 32 bits           │          │
│  │  EVENT TIMESTAMP         │  SUBSCRIBER COUNT        │          │
│  │                          │                          │          │
│  │  Monotonically           │  How many proxies are    │          │
│  │  increasing counter      │  currently reading this  │          │
│  │  (0 = invalid/writing)   │  slot                    │          │
│  └──────────────────────────┴──────────────────────────┘          │
│                                                                    │
│  States:                                                           │
│  ┌─────────────────────────────────────────────────────┐          │
│  │ timestamp=0, count=0  → INVALID (never written)     │          │
│  │ timestamp=0, count=1  → IN_WRITING (being filled)   │          │
│  │ timestamp>0, count=0  → READY (can be overwritten)  │          │
│  │ timestamp>0, count>0  → IN_USE (readers present)    │          │
│  └─────────────────────────────────────────────────────┘          │
│                                                                    │
│  All transitions via atomic compare-and-swap (CAS)                │
│  NO MUTEXES. NO KERNEL CALLS. PURE USERSPACE.                     │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/bindings/lola/event_slot_status.h](../../score/mw/com/impl/bindings/lola/event_slot_status.h)

---

## The Zero-Copy Write Path (Skeleton Side)

```
                          SKELETON PROCESS
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   1. skeleton.event_.Allocate()                                  │
│         │                                                        │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ ProviderEventDataControlLocalView   │                        │
│   │                                     │                        │
│   │  Find oldest/unused slot:           │                        │
│   │  - Skip slots with readers > 0     │                        │
│   │  - Pick slot with lowest timestamp  │                        │
│   │  - CAS: mark as IN_WRITING          │                        │
│   │                                     │                        │
│   │  Result: slot_index = 2             │                        │
│   └────────────────┬────────────────────┘                        │
│                    │                                              │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ Return SampleAllocateePtr<T>        │                        │
│   │                                     │                        │
│   │  Contains: pointer directly to      │                        │
│   │  Slot 2 in DATA shared memory       │                        │
│   └────────────────┬────────────────────┘                        │
│                    │                                              │
│         ▼                                                        │
│   2. sample->temperature = 42.5f;    ← WRITES TO SHARED MEMORY  │
│      sample->timestamp = now();        (NO COPY!)                │
│                                                                  │
│         │                                                        │
│         ▼                                                        │
│   3. skeleton.event_.Send(std::move(sample))                     │
│         │                                                        │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ CAS: Set timestamp to next value    │                        │
│   │ (marks slot as READY)               │                        │
│   │                                     │                        │
│   │ Notify subscribers via msg_passing  │                        │
│   └─────────────────────────────────────┘                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### SampleAllocateePtr — The Write Handle

```cpp
template <typename SampleType>
class SampleAllocateePtr {
    SampleType* managed_object_;           // Direct pointer to slot in SHM
    SlotIndexType event_slot_index_;        // Which slot (e.g., 2)
    EventDataControlComposite<> control_;  // Reference to control block

    // Destructor: if not Send()ed, releases the slot back
};
```

> **Related file**: [score/mw/com/impl/bindings/lola/sample_allocatee_ptr.h](../../score/mw/com/impl/bindings/lola/sample_allocatee_ptr.h)

---

## The Zero-Copy Read Path (Proxy Side)

```
                           PROXY PROCESS
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   1. proxy.event_.GetNewSamples(callback, max_count)             │
│         │                                                        │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ ConsumerEventDataControlLocalView   │                        │
│   │                                     │                        │
│   │  Scan all slots:                    │                        │
│   │  - Find slots with timestamp >      │                        │
│   │    last_seen_timestamp              │                        │
│   │  - For each new slot:              │                        │
│   │    CAS: increment subscriber_count  │                        │
│   │  - Sort by timestamp (oldest first) │                        │
│   └────────────────┬────────────────────┘                        │
│                    │                                              │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ Create SamplePtr<T> for each slot   │                        │
│   │                                     │                        │
│   │  Contains: const pointer directly   │                        │
│   │  to slot in DATA shared memory      │                        │
│   └────────────────┬────────────────────┘                        │
│                    │                                              │
│         ▼                                                        │
│   2. callback(sample_ptr) is invoked                             │
│         │                                                        │
│      auto& data = *sample_ptr;   ← READS FROM SHARED MEMORY     │
│      use(data.temperature);         (NO COPY!)                   │
│                                                                  │
│         │                                                        │
│         ▼                                                        │
│   3. SamplePtr goes out of scope (or is released)                │
│         │                                                        │
│         ▼                                                        │
│   ┌─────────────────────────────────────┐                        │
│   │ SlotDecrementer destructor:         │                        │
│   │ CAS: decrement subscriber_count     │                        │
│   │                                     │                        │
│   │ If count reaches 0 → slot is now    │                        │
│   │ available for skeleton to reuse     │                        │
│   └─────────────────────────────────────┘                        │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### SamplePtr — The Read Handle

```cpp
template <typename SampleType>
class SamplePtr {
    const SampleType* managed_object_;              // Direct CONST pointer to SHM
    std::optional<SlotDecrementer> slot_decrementer_; // Releases ref count on destroy
    
    // You can ONLY read, never write through SamplePtr
    const SampleType& operator*() const;
    const SampleType* operator->() const;
};
```

> **Related file**: [score/mw/com/impl/bindings/lola/sample_ptr.h](../../score/mw/com/impl/bindings/lola/sample_ptr.h)

---

## Memory Layout in Shared Memory

```
┌─────────────────────────────────────────────────────────────────────┐
│              DATA SHARED MEMORY OBJECT                                │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐     │
│  │ ServiceDataStorage (root object)                           │     │
│  │                                                           │     │
│  │  skeleton_pid_ = 1234                                     │     │
│  │  skeleton_uid_ = 1000                                     │     │
│  │                                                           │     │
│  │  events_ map:                                             │     │
│  │  ┌─────────────────────────────────────────────────┐     │     │
│  │  │ ElementFqId("temp", service_42)                  │     │     │
│  │  │   → OffsetPtr to:                               │     │     │
│  │  │     ┌──────┬──────┬──────┬──────┐              │     │     │
│  │  │     │ T[0] │ T[1] │ T[2] │ T[3] │ (4 slots)   │     │     │
│  │  │     └──────┴──────┴──────┴──────┘              │     │     │
│  │  ├─────────────────────────────────────────────────┤     │     │
│  │  │ ElementFqId("speed", service_42)                │     │     │
│  │  │   → OffsetPtr to:                               │     │     │
│  │  │     ┌──────┬──────┬──────┬──────┬──────┐       │     │     │
│  │  │     │ S[0] │ S[1] │ S[2] │ S[3] │ S[4] │(5)   │     │     │
│  │  │     └──────┴──────┴──────┴──────┴──────┘       │     │     │
│  │  └─────────────────────────────────────────────────┘     │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│              CONTROL SHARED MEMORY OBJECT                             │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────┐     │
│  │ ServiceDataControl (root object)                           │     │
│  │                                                           │     │
│  │  event_controls_ map:                                     │     │
│  │  ┌─────────────────────────────────────────────────┐     │     │
│  │  │ ElementFqId("temp")                              │     │     │
│  │  │   → EventControl:                               │     │     │
│  │  │     ┌────────────────────────────────────┐      │     │     │
│  │  │     │ EventDataControl (slot statuses):   │      │     │     │
│  │  │     │  [atomic64][atomic64][atomic64][a64]│      │     │     │
│  │  │     ├────────────────────────────────────┤      │     │     │
│  │  │     │ EventSubscriptionControl:           │      │     │     │
│  │  │     │  max_subscribers, active count      │      │     │     │
│  │  │     ├────────────────────────────────────┤      │     │     │
│  │  │     │ TransactionLogSet:                  │      │     │     │
│  │  │     │  per-subscriber write tracking      │      │     │     │
│  │  │     └────────────────────────────────────┘      │     │     │
│  │  └─────────────────────────────────────────────────┘     │     │
│  │                                                           │     │
│  │  application_id_pid_mapping_:                             │     │
│  │    Maps application IDs → PIDs (for crash detection)      │     │
│  └───────────────────────────────────────────────────────────┘     │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Offset Pointers — Cross-Process Addressing

Normal C++ pointers don't work across processes (each process has different virtual address space). LoLa uses **offset pointers**:

```
Process A (Skeleton):              Process B (Proxy):
Virtual address: 0x7F000000        Virtual address: 0x3A000000
                                   
┌─────────────────────┐           ┌─────────────────────┐
│ SHM mapped at       │           │ SHM mapped at       │
│ 0x7F000000          │           │ 0x3A000000          │
│                     │           │                     │
│ Slot[2] at          │           │ Slot[2] at          │
│ 0x7F000100          │           │ 0x3A000100          │
│                     │           │                     │
│ Offset from base:   │           │ Offset from base:   │
│ +0x100 (SAME!)      │           │ +0x100 (SAME!)      │
└─────────────────────┘           └─────────────────────┘

OffsetPtr stores: +0x100 (offset from SHM base)
→ Works in ANY process that maps the same SHM!
```

> LoLa uses Boost.Interprocess offset pointers + PMR (Polymorphic Memory Resources) allocators.

---

## SkeletonMemoryManager — Setup & Lifecycle

```
┌─────────────────────────────────────────────────────────┐
│ SkeletonMemoryManager                                    │
│                                                         │
│ CreateSharedMemory()                                    │
│   │                                                     │
│   ├── 1. shm_open() + mmap() for Data SHM             │
│   ├── 2. shm_open() + mmap() for Control SHM          │
│   ├── 3. Construct ServiceDataStorage in Data SHM      │
│   ├── 4. Construct ServiceDataControl in Control SHM   │
│   └── 5. For each event configured:                    │
│       ├── Allocate N slots in Data SHM                 │
│       └── Create N atomic statuses in Control SHM      │
│                                                         │
│ OpenExistingSharedMemory()                              │
│   │ (Used when proxy connects, or partial restart)     │
│   ├── 1. Open existing SHM by name                     │
│   └── 2. Map into this process's address space         │
│                                                         │
│ CleanupSharedMemoryAfterCrash()                        │
│   │ (Detect via PID mapping)                           │
│   ├── 1. Reset all slots owned by crashed process      │
│   └── 2. Decrement subscriber counts                   │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/bindings/lola/skeleton_memory_manager.h](../../score/mw/com/impl/bindings/lola/skeleton_memory_manager.h)

---

## Complete Data Flow — End to End

```
SKELETON                    SHARED MEMORY                    PROXY
════════                    ═════════════                    ═════

Allocate()
    │
    ├─ Find slot with      ┌─────────────┐
    │  lowest timestamp    │  Slot 2:    │
    │  and 0 readers       │  Status:    │
    │                      │  [0|0]      │ ← invalid
    ├─ CAS: mark writing   │  [0|1]      │ ← in_writing
    │                      │             │
    ├─ Return ptr to ──────┤► Data[2]    │
    │  SampleAllocateePtr  │             │
    │                      │             │
Write data                 │             │
    │                      │             │
    ├─ *ptr = my_data  ────┤► Data[2] ←──── actual bytes written
    │                      │             │
Send()                     │             │
    │                      │             │
    ├─ CAS: set timestamp  │  [5|0]      │ ← ready, ts=5
    │                      │             │
    ├─ Notify via msg      │             │          GetNewSamples()
    │  passing channel     │             │               │
    │                      │             │     Scan all slots for
    │                      │             │     timestamp > last_seen
    │                      │             │               │
    │                      │             │     CAS: increment readers
    │                      │  [5|1]      │ ← 1 reader    │
    │                      │             │               │
    │                      │  Data[2] ───┤──────────► SamplePtr
    │                      │             │           (const pointer)
    │                      │             │               │
    │                      │             │     callback(*ptr)
    │                      │             │     reads data directly
    │                      │             │               │
    │                      │             │     ~SamplePtr()
    │                      │             │     CAS: decrement readers
    │                      │  [5|0]      │ ← 0 readers (reusable)
    │                      └─────────────┘
```

---

## Why This Design Is Fast

| Traditional IPC | LoLa Zero-Copy |
|----------------|----------------|
| App writes to local buffer | App writes directly to SHM slot |
| Copy #1: app → kernel | _(no copy)_ |
| Copy #2: kernel → receiver | _(no copy)_ |
| Receiver reads from local buffer | Receiver reads directly from SHM slot |
| **2+ copies, 2+ syscalls** | **0 copies, 0 syscalls for data** |

The only syscall involvement is for **notifications** ("new data available"), handled by the message passing layer (Part 6).

---

## Key Source Files

| File | Purpose |
|------|---------|
| [event_data_storage.h](../../score/mw/com/impl/bindings/lola/event_data_storage.h) | Per-event data slots in SHM |
| [event_data_control.h](../../score/mw/com/impl/bindings/lola/event_data_control.h) | Per-event atomic control slots |
| [event_slot_status.h](../../score/mw/com/impl/bindings/lola/event_slot_status.h) | 64-bit atomic status packing |
| [event_control.h](../../score/mw/com/impl/bindings/lola/event_control.h) | Combines data control + subscription |
| [service_data_storage.h](../../score/mw/com/impl/bindings/lola/service_data_storage.h) | Root anchor in Data SHM |
| [service_data_control.h](../../score/mw/com/impl/bindings/lola/service_data_control.h) | Root anchor in Control SHM |
| [skeleton_memory_manager.h](../../score/mw/com/impl/bindings/lola/skeleton_memory_manager.h) | SHM creation & lifecycle |
| [sample_ptr.h](../../score/mw/com/impl/bindings/lola/sample_ptr.h) | Zero-copy read pointer |
| [sample_allocatee_ptr.h](../../score/mw/com/impl/bindings/lola/sample_allocatee_ptr.h) | Zero-copy write pointer |
| [provider_event_data_control_local_view.h](../../score/mw/com/impl/bindings/lola/provider_event_data_control_local_view.h) | Slot allocation logic |
| [consumer_event_data_control_local_view.h](../../score/mw/com/impl/bindings/lola/consumer_event_data_control_local_view.h) | Slot reading logic |
| [slot_collector.h](../../score/mw/com/impl/bindings/lola/slot_collector.h) | Finds new samples |
| [slot_decrementer.h](../../score/mw/com/impl/bindings/lola/slot_decrementer.h) | RAII reference count release |
| [score/mw/com/design/shared_mem_layout/](../../score/mw/com/design/shared_mem_layout/) | Design documentation |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **Slot** | Pre-allocated position in SHM; one slot = one sample |
| **EventSlotStatus** | 64-bit atomic: upper=timestamp, lower=reader_count |
| **SampleAllocateePtr** | Direct write pointer into SHM (returned by Allocate) |
| **SamplePtr** | Direct read pointer into SHM (returned by GetNewSamples) |
| **Offset pointer** | Stores offset from SHM base, works across processes |
| **Two SHMs** | Data (samples) + Control (atomics) — cache separation |
| **Lock-free** | All synchronization via atomic CAS, no mutexes |

---

## Next: [Part 4 — Event System →](part4_event_system.md)

How do SkeletonEvent and ProxyEvent wrap this machinery? What about subscriptions, receive handlers, and the state machine?

---

## Previous: [← Part 2 — Service Discovery](part2_service_discovery.md)
