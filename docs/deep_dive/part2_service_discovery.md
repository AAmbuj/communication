# Part 2: Service Discovery — How Services Find Each Other

## Overview

Service Discovery is the mechanism that connects **Skeletons** (publishers) with **Proxies** (subscribers). Without it, a proxy wouldn't know where the skeleton's shared memory lives.

LoLa uses a **flag-file-based** discovery mechanism — simple files on a tmpfs filesystem that act as registration entries.

---

## The Big Picture

```
┌────────────────────────────────────────────────────────────────────────────┐
│                        SERVICE DISCOVERY FLOW                                │
│                                                                            │
│   SKELETON (Publisher)                     PROXY (Subscriber)               │
│                                                                            │
│   1. OfferService()                                                        │
│         │                                                                  │
│         ▼                                                                  │
│   ┌─────────────┐                                                          │
│   │ Create Flag │     ┌──────────────────────────────┐                     │
│   │ File on     │────►│  /tmp/mw_com_lola/           │                     │
│   │ Filesystem  │     │    └── <service_id>/         │                     │
│   └─────────────┘     │        └── <instance_id>/    │     3. FindService()│
│                        │            └── 1234_asil-b_7 │◄───────────┤        │
│                        └──────────────────────────────┘            │        │
│                                       ▲                           │        │
│                                       │                    ┌──────┴──────┐ │
│                        2. inotify watches directory        │ Crawl dirs  │ │
│                                                            │ for flags   │ │
│                                                            └─────────────┘ │
│                                                                    │        │
│                                                            4. Match found!  │
│                                                                    │        │
│                                                            5. Return Handle │
│                                                                    ▼        │
│                                                            Proxy::Create()  │
└────────────────────────────────────────────────────────────────────────────┘
```

---

## The Identifier Chain (Design → Runtime)

There are **three levels of identifiers** that progressively get more specific:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  DESIGN TIME                    CONFIGURATION               RUNTIME      │
│                                                                          │
│  ┌─────────────────┐           ┌──────────────┐        ┌────────────┐  │
│  │ InstanceSpecifier│──resolve──►│InstanceIdent-│──map──►│LolaService │  │
│  │                 │   (JSON    │ ifier        │  to    │InstanceId  │  │
│  │"my_app/sensor"  │  manifest) │              │ LoLa   │            │  │
│  └─────────────────┘           │ service_id:42│        │ sid:42     │  │
│                                │ instance_id:1│        │ iid:1      │  │
│  What YOU write in             └──────────────┘        └────────────┘  │
│  your application code         What's in the JSON      What's on the   │
│                                config file              filesystem      │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### InstanceSpecifier (Your code)
- A human-readable string like `"my_app/temperature_sensor"`
- Defined at design time in the application model
- Maps to deployment config via JSON manifest

### InstanceIdentifier (Resolved from config)
- Contains: `service_id`, `instance_id`, `version`, `binding_type`, `quality_type`
- Resolved from InstanceSpecifier using the JSON manifest
- Multiple InstanceIdentifiers can map to one InstanceSpecifier (different deployments)

### LolaServiceInstanceIdentifier (On filesystem)
- The final form used to create/find flag files
- Maps directly to the directory path: `<service_id>/<instance_id>/`

> **Related files**:
> - [score/mw/com/impl/instance_specifier.h](../../score/mw/com/impl/instance_specifier.h)
> - [score/mw/com/impl/instance_identifier.h](../../score/mw/com/impl/instance_identifier.h)
> - [score/mw/com/impl/bindings/lola/service_discovery/lola_service_instance_identifier.h](../../score/mw/com/impl/bindings/lola/service_discovery/lola_service_instance_identifier.h)

---

## Flag Files — The Registration Mechanism

### What is a Flag File?

A flag file is simply an **empty file** whose **name and path** carry all the information:

```
Path format:
<sd>/mw_com_lola/<service_id>/<instance_id>/<PID>_<asil>_<unique_seed>

Example:
/tmp/mw_com_lola/42/1/1234_asil-b_7
                 │  │  │    │      │
                 │  │  │    │      └─ Unique seed (changes each re-offer)
                 │  │  │    └──────── ASIL level (asil-b or asil-qm)
                 │  │  └───────────── Process ID of the offering skeleton
                 │  └──────────────── Instance ID
                 └─────────────────── Service ID
```

### Why This Design?

1. **Simple** — No database, no daemon, just files
2. **Crash-safe** — If a process crashes, its flag files become stale (detectable via PID)
3. **Watch-friendly** — Linux `inotify` can efficiently watch for new/deleted files
4. **Permission-safe** — Standard filesystem permissions control access
5. **Non-persistent** — Lives on tmpfs, auto-cleaned on reboot

> **Related file**: [score/mw/com/impl/bindings/lola/service_discovery/flag_file.h](../../score/mw/com/impl/bindings/lola/service_discovery/flag_file.h)

---

## OfferService() — Skeleton Side

When a skeleton calls `OfferService()`, here's what happens:

```
OfferService()
     │
     ▼
┌────────────────────────────────┐
│ 1. Resolve InstanceSpecifier    │
│    to InstanceIdentifier        │
│    (using JSON manifest)        │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 2. Create directory structure   │
│    /tmp/mw_com_lola/42/1/      │
│    (service_id/instance_id)     │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 3. Create flag file             │
│    1234_asil-b_7                │
│    (PID_quality_seed)           │
│                                 │
│    Permissions:                 │
│    - Owner: read/write          │
│    - Others: read only          │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 4. Set up shared memory         │
│    (LoLa binding creates SHM    │
│     regions for data exchange)   │
└────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/service_discovery.h](../../score/mw/com/impl/service_discovery.h) — `ServiceDiscovery::OfferService()`

---

## FindService() — Proxy Side

When a proxy calls `FindService()`, here's the process:

```
FindService() / StartFindService()
     │
     ▼
┌────────────────────────────────┐
│ 1. Resolve InstanceSpecifier    │
│    to know which service_id     │
│    and instance_id to look for  │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 2. Set up inotify watch         │
│                                 │
│    Specific instance:           │
│    Watch: /tmp/.../42/1/        │
│                                 │
│    Any instance:                │
│    Watch: /tmp/.../42/          │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 3. Crawl existing directories   │
│    (FlagFileCrawler)            │
│                                 │
│    Scan for existing flag files │
│    that match our criteria      │
└───────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────┐
│ 4. Found? Return HandleType     │
│                                 │
│    Handle contains:             │
│    - InstanceIdentifier         │
│    - Connection info to SHM     │
│    - Quality type               │
└────────────────────────────────┘
```

### Two Modes of Finding

| Mode | Method | Behavior |
|------|--------|----------|
| **One-shot** | `FindService()` | Synchronous. Returns what's available NOW. |
| **Continuous** | `StartFindService()` | Async. Calls your callback whenever services appear/disappear. |

```cpp
// One-shot: "Is the service there RIGHT NOW?"
auto handles = MyProxy::FindService(instance_spec);

// Continuous: "Tell me whenever the service appears or disappears"
auto find_handle = MyProxy::StartFindService(
    [](ServiceHandleContainer<HandleType> handles) {
        // Called each time service availability changes
        for (auto& handle : handles) {
            auto proxy = MyProxy::Create(std::move(handle));
            // ... use proxy ...
        }
    },
    instance_spec
);

// Later: stop watching
MyProxy::StopFindService(find_handle);
```

---

## The FlagFileCrawler — Filesystem Scanner

The `FlagFileCrawler` is the component that actually reads the filesystem:

```
FlagFileCrawler
     │
     ├── Crawl()           → Scan directories, return found instances
     │
     ├── CrawlAndWatch()   → Scan + set up inotify watches for future changes
     │
     └── CrawlAndWatchWithRetry() → Same but with retry logic for race conditions
```

It handles:
- Parsing flag file names to extract PID, ASIL level, and seed
- Grouping results by quality type (ASIL-B vs QM)
- Setting up `inotify` watches at the correct directory level

> **Related file**: [score/mw/com/impl/bindings/lola/service_discovery/flag_file_crawler.h](../../score/mw/com/impl/bindings/lola/service_discovery/flag_file_crawler.h)

---

## Quality Types (ASIL-B vs QM)

Services can be offered with different safety qualification levels:

```
┌────────────────────────────────────────────────────────┐
│                                                        │
│  ASIL-B (Safety Critical)     QM (Quality Management) │
│                                                        │
│  • Braking system             • Infotainment           │
│  • Steering control           • Music player           │
│  • Collision detection        • Navigation display     │
│                                                        │
│  Flag file: 1234_asil-b_7     Flag file: 1234_asil-qm_3│
│                                                        │
└────────────────────────────────────────────────────────┘
```

A skeleton can offer the **same service** at both quality levels simultaneously (separate flag files for each).

> **Related file**: [score/mw/com/impl/bindings/lola/service_discovery/quality_aware_container.h](../../score/mw/com/impl/bindings/lola/service_discovery/quality_aware_container.h)

---

## Thread Safety & Worker Thread

Service Discovery uses a **dedicated worker thread** with these guarantees:

```
┌────────────────────────────────────────────────────────────┐
│                                                            │
│  Main Thread                    SD Worker Thread           │
│                                                            │
│  OfferService() ──mutex──►  Process flag file creation     │
│  FindService()  ──mutex──►  Crawl + Watch                  │
│  StopFindService() ─────►  Remove watches                  │
│                                                            │
│  User callbacks are invoked FROM the worker thread         │
│  (or synchronously during StartFindService for             │
│   already-existing services)                               │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

---

## Configuration (JSON Manifest)

The mapping from `InstanceSpecifier` to actual deployment is defined in a JSON file:

```json
{
  "serviceInstances": {
    "my_app/temperature_sensor": {
      "serviceId": 42,
      "instanceId": 1,
      "version": { "major": 1, "minor": 0 },
      "binding": "lola",
      "quality": "asil-b"
    }
  }
}
```

This file is loaded at runtime via:
```bash
./my_app -service_instance_manifest /path/to/config.json
```

Or placed at the default manifest path.

> **Related files**:
> - [score/mw/com/impl/configuration/](../../score/mw/com/impl/configuration/) — Configuration parsing
> - [score/mw/com/example/ipc_bridge/etc/](../../score/mw/com/example/ipc_bridge/etc/) — Example config files

---

## Key Source Files for Service Discovery

| File | Purpose |
|------|---------|
| [service_discovery.h](../../score/mw/com/impl/service_discovery.h) | Top-level SD class (binding-independent) |
| [i_service_discovery.h](../../score/mw/com/impl/i_service_discovery.h) | Interface for SD |
| [flag_file.h](../../score/mw/com/impl/bindings/lola/service_discovery/flag_file.h) | Flag file creation/deletion |
| [flag_file_crawler.h](../../score/mw/com/impl/bindings/lola/service_discovery/flag_file_crawler.h) | Scanning directories for flag files |
| [lola_service_instance_identifier.h](../../score/mw/com/impl/bindings/lola/service_discovery/lola_service_instance_identifier.h) | LoLa-specific service ID |
| [known_instances_container.h](../../score/mw/com/impl/bindings/lola/service_discovery/known_instances_container.h) | Tracks discovered instances |
| [find_service_handler.h](../../score/mw/com/impl/find_service_handler.h) | Callback type for async find |
| [instance_specifier.h](../../score/mw/com/impl/instance_specifier.h) | Design-time port name |
| [instance_identifier.h](../../score/mw/com/impl/instance_identifier.h) | Runtime service identifier |
| [score/mw/com/design/service_discovery/](../../score/mw/com/design/service_discovery/) | Design documentation + UML |

---

## Summary

| Step | What Happens | Who Does It |
|------|-------------|-------------|
| 1 | Skeleton calls `OfferService()` | `ServiceDiscovery` |
| 2 | Flag file created on filesystem | `FlagFile::Make()` |
| 3 | Proxy calls `FindService()` or `StartFindService()` | `ServiceDiscovery` |
| 4 | `FlagFileCrawler` scans directories | `FlagFileCrawler::Crawl()` |
| 5 | inotify watches set for future changes | `FlagFileCrawler::CrawlAndWatch()` |
| 6 | Match found → `HandleType` returned to proxy | `ServiceDiscovery` |
| 7 | Proxy creates connection using handle | `Proxy::Create(handle)` |

---

## Next: [Part 3 — Shared Memory & Zero-Copy →](part3_shared_memory.md)

What happens AFTER the proxy finds the skeleton? How does the shared memory get set up? How does zero-copy actually work?

---

## Previous: [← Part 1 — Fundamentals](part1_fundamentals.md)
