# Part 7: Safety & Transactions — Crash Recovery and ASIL-B

## Overview

LoLa is designed for **safety-critical automotive systems** (ASIL-B qualified). This means it must handle:
- Process crashes at ANY point (including mid-write to shared memory)
- Mixed-criticality (ASIL-B and QM processes sharing infrastructure)
- Partial restarts (one process restarts while others continue)
- No deadlocks, no resource leaks, bounded recovery time

---

## The Problem: Crashes in Shared Memory

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  WHAT CAN GO WRONG:                                                    │
│                                                                        │
│  1. Proxy crashes while incrementing reference count on a slot         │
│     → Slot permanently "in use" → skeleton can never reuse it         │
│                                                                        │
│  2. Skeleton crashes while writing to a slot (IN_WRITING)             │
│     → Slot permanently locked → no new data can be published          │
│                                                                        │
│  3. Proxy crashes after subscribing but before unsubscribing          │
│     → Subscription count permanently elevated → resource leak         │
│                                                                        │
│  4. Proxy crashes mid-method-call                                     │
│     → Call queue slot stuck "in use" → method can't be called again   │
│                                                                        │
│  WITHOUT SAFETY MECHANISMS:                                            │
│  Each crash permanently degrades the system until reboot!              │
│                                                                        │
│  WITH TRANSACTION LOGS:                                                │
│  Crashed operations are detected and rolled back on restart.          │
│  System returns to clean state.                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Transaction Logs — The Crash Recovery Mechanism

### What Is a Transaction Log?

A transaction log records **what a proxy is about to do** BEFORE it does it. If the proxy crashes, the next process that starts can read the log and **undo** the incomplete operation.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  NORMAL OPERATION (no crash):                                           │
│                                                                         │
│  1. Write to transaction log: "I'm about to increment slot 3"          │
│  2. Actually increment slot 3 reference count (atomic CAS)             │
│  3. Write to transaction log: "Done, operation complete"                │
│                                                                         │
│  ─────────────────────────────────────────────────────────────────────  │
│                                                                         │
│  CRASH DURING OPERATION:                                                │
│                                                                         │
│  1. Write to transaction log: "I'm about to increment slot 3"          │
│  2. Actually increment slot 3 reference count (atomic CAS)             │
│  3. *** CRASH ***                                                       │
│                                                                         │
│  ON RESTART:                                                            │
│  4. Read transaction log: "started increment slot 3, NOT completed"    │
│  5. ROLLBACK: decrement slot 3 reference count                         │
│  6. Slot 3 is clean again ✓                                           │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Transaction Log Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│  TransactionLog (one per proxy per event)                            │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────┐    │
│  │ reference_count_slots_: [TransactionLogSlot per data slot] │    │
│  │                                                            │    │
│  │  Slot 0: [begin: false, end: NONE]    ← no operation      │    │
│  │  Slot 1: [begin: true,  end: DONE]    ← completed OK      │    │
│  │  Slot 2: [begin: true,  end: NONE]    ← CRASHED! ROLLBACK │    │
│  │  Slot 3: [begin: false, end: NONE]    ← no operation      │    │
│  │                                                            │    │
│  ├────────────────────────────────────────────────────────────┤    │
│  │ subscribe_transactions_: TransactionLogSlot                │    │
│  │                                                            │    │
│  │  [begin: false, end: NONE]  ← no pending subscription     │    │
│  │                                                            │    │
│  ├────────────────────────────────────────────────────────────┤    │
│  │ subscription_max_sample_count_: optional<5>                │    │
│  │                                                            │    │
│  │  Remembered so rollback knows what to undo                 │    │
│  └────────────────────────────────────────────────────────────┘    │
│                                                                     │
│  STORED IN: Control Shared Memory (survives process crash!)         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

> **Related files**:
> - [score/mw/com/impl/bindings/lola/transaction_log.h](../../score/mw/com/impl/bindings/lola/transaction_log.h)
> - [score/mw/com/impl/bindings/lola/transaction_log_slot.h](../../score/mw/com/impl/bindings/lola/transaction_log_slot.h)

---

## TransactionLogSet — Managing Multiple Proxies

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  EVENT CONTROL (in shared memory)                                       │
│                                                                         │
│  TransactionLogSet (one per event, sized for max_subscribers):          │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────┐       │
│  │ Slot 0: [TransactionLog for Proxy A]  app_id=100, pid=1234 │       │
│  │ Slot 1: [TransactionLog for Proxy B]  app_id=200, pid=5678 │       │
│  │ Slot 2: [empty — available for next subscriber]             │       │
│  │ Slot 3: [empty]                                             │       │
│  └─────────────────────────────────────────────────────────────┘       │
│                                                                         │
│  ALLOCATION: Lock-free atomic compare_exchange                          │
│  • Proxy does CAS on transaction_log_id_ field to claim a slot         │
│  • If CAS fails → retry next slot                                     │
│  • No mutex, works across processes                                    │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/bindings/lola/transaction_log_set.h](../../score/mw/com/impl/bindings/lola/transaction_log_set.h)

---

## Crash Detection: ApplicationIdPidMapping

How does the system know a process crashed? Via PID changes:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  APPLICATION ID → PID MAPPING (in shared memory)                        │
│                                                                         │
│  ┌────────────────────────────────────────────────────┐                │
│  │ app_id: 100 → pid: 1234                            │                │
│  │ app_id: 200 → pid: 5678                            │                │
│  └────────────────────────────────────────────────────┘                │
│                                                                         │
│  SCENARIO: Process with app_id=100 crashes and restarts                │
│                                                                         │
│  New process registers: app_id=100, pid=9999                           │
│                                                                         │
│  RegisterPid() returns: old_pid=1234                                   │
│  Current pid=9999 ≠ old_pid=1234                                       │
│                                                                         │
│  → CRASH DETECTED!                                                      │
│  → Notify skeleton: "PID 1234 is dead, cleanup its artifacts"          │
│  → Skeleton removes message passing connections for PID 1234           │
│  → Transaction log rollback begins                                     │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/bindings/lola/application_id_pid_mapping.h](../../score/mw/com/impl/bindings/lola/application_id_pid_mapping.h)

---

## Rollback Execution — Two Phases

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  PHASE 1: PrepareRollback() — On Proxy Restart                         │
│                                                                         │
│  ┌─────────────────────────────────────────────┐                       │
│  │ 1. Register PID with app_id                  │                       │
│  │    (detects if previous incarnation crashed) │                       │
│  │                                              │                       │
│  │ 2. If crash detected:                        │                       │
│  │    Notify skeleton of dead PID               │                       │
│  │                                              │                       │
│  │ 3. Mark transaction logs as "needs rollback" │                       │
│  │    (only logs from BEFORE restart)           │                       │
│  └─────────────────────────────────────────────┘                       │
│                                                                         │
│  PHASE 2: RollbackTransactionLogs() — Undo Incomplete Ops              │
│                                                                         │
│  ┌─────────────────────────────────────────────┐                       │
│  │ For each event in the service:              │                       │
│  │                                              │                       │
│  │   For each slot with begin=true, end≠DONE:  │                       │
│  │     → DereferenceEvent(slot_index)           │                       │
│  │       (undo the reference count increment)   │                       │
│  │                                              │                       │
│  │   If subscribe transaction incomplete:       │                       │
│  │     → Unsubscribe(max_sample_count)          │                       │
│  │       (undo the subscription)                │                       │
│  │                                              │                       │
│  │   Clear the transaction log                  │                       │
│  └─────────────────────────────────────────────┘                       │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/bindings/lola/transaction_log_rollback_executor.h](../../score/mw/com/impl/bindings/lola/transaction_log_rollback_executor.h)

---

## Partial Restart — Process Restarts While Others Continue

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  PARTIAL RESTART COORDINATION (via marker files + file locks)           │
│                                                                         │
│  Filesystem:                                                            │
│  /tmp/lola/42/1/                                                       │
│    ├── service_instance_existing.marker    ← skeleton's exclusive lock  │
│    └── service_instance_usage.marker       ← proxy shared lock          │
│                                                                         │
│  ════════════════════════════════════════════════════════════════════    │
│                                                                         │
│  PROXY RESTART:                                                         │
│                                                                         │
│  1. Acquire SHARED lock on usage.marker                                │
│     • Success → skeleton is alive, safe to proceed                     │
│     • Fail → skeleton not ready yet, wait/retry                        │
│                                                                         │
│  2. Rollback transaction logs (Phase 1 + Phase 2)                      │
│                                                                         │
│  3. Create new subscription (fresh start)                              │
│                                                                         │
│  ════════════════════════════════════════════════════════════════════    │
│                                                                         │
│  SKELETON RESTART:                                                      │
│                                                                         │
│  1. Acquire EXCLUSIVE lock on existing.marker                          │
│     • Ensures only one skeleton per service instance                   │
│                                                                         │
│  2. Try to acquire lock on usage.marker                                │
│     ┌─────────────────────────────────────────────────┐                │
│     │ Lock SUCCESS → No active proxies                 │                │
│     │   → Destroy old shared memory                    │                │
│     │   → Create fresh shared memory                   │                │
│     │   → Clean start                                  │                │
│     ├─────────────────────────────────────────────────┤                │
│     │ Lock FAIL → Active proxies still connected       │                │
│     │   → REUSE existing shared memory                 │                │
│     │   → Clean up IN_WRITING slots                    │                │
│     │   → Resume offering with existing data           │                │
│     └─────────────────────────────────────────────────┘                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## ASIL-B Compliance — Memory Isolation

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  MIXED-CRITICALITY SHARED MEMORY ARCHITECTURE                           │
│                                                                         │
│  Per service instance, SEPARATE segments per ASIL level:               │
│                                                                         │
│  ┌───────────────────────────────────┐                                 │
│  │  DATA Shared Memory (QM)          │ ← Actual sample data            │
│  │  • Read by ASIL-B proxies         │                                 │
│  │  • Read by QM proxies             │                                 │
│  │  • Written by skeleton only       │                                 │
│  └───────────────────────────────────┘                                 │
│                                                                         │
│  ┌───────────────────────────────────┐                                 │
│  │  CONTROL Shared Memory (ASIL-B)   │ ← Safety-critical control       │
│  │  • Slot states for ASIL-B proxies │                                 │
│  │  • Cannot be corrupted by QM code │                                 │
│  │  • Separate memory permissions    │                                 │
│  └───────────────────────────────────┘                                 │
│                                                                         │
│  ┌───────────────────────────────────┐                                 │
│  │  CONTROL Shared Memory (QM)       │ ← Non-safety control            │
│  │  • Slot states for QM proxies     │                                 │
│  │  • If corrupted: only QM affected │                                 │
│  │  • ASIL-B path unaffected         │                                 │
│  └───────────────────────────────────┘                                 │
│                                                                         │
│  GUARANTEE: QM process corruption CANNOT affect ASIL-B operation       │
│                                                                         │
│  SKELETON WRITE STRATEGY:                                              │
│  1. Mark slot IN_WRITING in ASIL-B control                             │
│  2. Try to mark same slot in QM control                                │
│  3. If QM slot unavailable:                                            │
│     → Undo ASIL-B slot, retry with different slot                     │
│     → Bounded retries to detect QM corruption                         │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## What Happens When Things Crash

### Scenario 1: Proxy Crashes Mid-Reference-Count

```
BEFORE:  Slot 3 status = [timestamp:5 | readers:2]
CRASH:   Proxy B incremented to readers:3, then crashed
RESULT:  Slot 3 status = [timestamp:5 | readers:3]  ← LEAK!

ON RESTART:
  Transaction log shows: "Proxy B began increment on slot 3, not completed"
  Rollback: Decrement readers on slot 3
  AFTER:   Slot 3 status = [timestamp:5 | readers:2]  ← FIXED ✓
```

### Scenario 2: Skeleton Crashes Mid-Write

```
BEFORE:  Slot 2 status = [timestamp:0 | readers:1]  ← IN_WRITING
CRASH:   Skeleton crashed while writing data

ON SKELETON RESTART:
  1. Try exclusive lock on usage marker → FAILS (proxies connected)
  2. Reuse existing shared memory
  3. Scan all slots: find slot 2 is IN_WRITING
  4. Reset slot 2 to [timestamp:0 | readers:0] (INVALID)
  5. Resume normal operation
```

### Scenario 3: Proxy Crashes After Subscribe

```
BEFORE:  EventSubscriptionControl: active_subscribers = 3
CRASH:   Proxy C subscribed with max_count=5, then crashed

ON RESTART:
  Transaction log shows: "subscribe with max_count=5 began, not completed"
  Rollback: Unsubscribe(max_count=5)
  AFTER:   active_subscribers = 2  ← FIXED ✓
```

---

## Safety Design Principles

| Principle | How LoLa Implements It |
|-----------|----------------------|
| **No deadlocks** | Lock-free atomics (no mutexes in shared memory) |
| **Bounded recovery** | Transaction logs have fixed size; rollback is O(n_slots) |
| **ASIL isolation** | Separate control SHM per quality level |
| **Crash tolerance** | Transaction logs + marker files + PID mapping |
| **No resource leaks** | OS auto-releases file locks on process death |
| **Deterministic** | No dynamic allocation after init; bounded retry counts |
| **Idempotent recovery** | Rolling back twice has no effect |

---

## Key Source Files

| File | Purpose |
|------|---------|
| [transaction_log.h](../../score/mw/com/impl/bindings/lola/transaction_log.h) | Per-proxy transaction log |
| [transaction_log_slot.h](../../score/mw/com/impl/bindings/lola/transaction_log_slot.h) | Single operation record |
| [transaction_log_set.h](../../score/mw/com/impl/bindings/lola/transaction_log_set.h) | All logs for one event |
| [transaction_log_rollback_executor.h](../../score/mw/com/impl/bindings/lola/transaction_log_rollback_executor.h) | Rollback logic |
| [rollback_synchronization.h](../../score/mw/com/impl/bindings/lola/rollback_synchronization.h) | Process-local mutex |
| [application_id_pid_mapping.h](../../score/mw/com/impl/bindings/lola/application_id_pid_mapping.h) | Crash detection |
| [score/mw/com/design/partial_restart/](../../score/mw/com/design/partial_restart/) | Design documentation |
| [score/mw/com/dependability/](../../score/mw/com/dependability/) | Safety architecture |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **Transaction Log** | Records what proxy is about to do; enables undo on crash |
| **TransactionLogSet** | One per event; manages logs for all subscribed proxies |
| **PID Mapping** | Detects crashed processes by PID change on restart |
| **Rollback** | Undoes incomplete operations (ref count, subscriptions) |
| **Partial Restart** | Process restarts while others continue; marker file locks coordinate |
| **ASIL-B Isolation** | Separate control SHM prevents QM→ASIL interference |
| **Lock-free** | No mutexes in shared memory; atomics + bounded retries |
| **Marker files** | File locks coordinate skeleton/proxy lifecycle |

---

## Next: [Part 8 — Configuration & Deployment →](part8_configuration.md)

How is LoLa configured? What's in the JSON manifest? How do service IDs get assigned?

---

## Previous: [← Part 6 — Message Passing Layer](part6_message_passing.md)
