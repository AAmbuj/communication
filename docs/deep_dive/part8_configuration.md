# Part 8: Configuration & Deployment — JSON Manifests and Runtime Setup

## Overview

LoLa separates **application code** from **deployment details**. Your code uses `InstanceSpecifier` (a port name), and a **JSON manifest** maps that to actual service IDs, buffer sizes, permissions, etc. This means the same code can run with different deployments without recompilation.

---

## Configuration Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  YOUR CODE                     JSON MANIFEST              RUNTIME       │
│                                                                         │
│  InstanceSpecifier             mw_com_config.json         Shared Memory │
│  "my_app/sensor"     ────────► serviceId: 42      ──────► /tmp/lola/42/ │
│                                instanceId: 1              4 slots       │
│                                numberOfSampleSlots: 4    max 5 subs    │
│                                maxSubscribers: 5                        │
│                                asil-level: "B"                          │
│                                                                         │
│  Code is UNCHANGED             Config changes per         Runtime adapts│
│  across deployments            vehicle variant/ECU                      │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## JSON Manifest Structure

The configuration has **4 sections**:

```json
{
  "serviceTypes": [...],        // 1. Service interface definitions
  "serviceInstances": [...],    // 2. Deployment bindings
  "global": {...},              // 3. Process-level settings
  "tracing": {...}              // 4. Tracing configuration
}
```

---

### Section 1: serviceTypes — Define the Interface

```json
"serviceTypes": [
  {
    "serviceTypeName": "/score/services/TemperatureService",
    "version": {"major": 1, "minor": 0},
    "bindings": [
      {
        "binding": "SHM",
        "serviceId": 42,
        "events": [
          {"eventName": "Temperature", "eventId": 1},
          {"eventName": "Humidity", "eventId": 2}
        ],
        "fields": [
          {"fieldName": "SensorStatus", "fieldId": 10}
        ],
        "methods": [
          {"methodName": "Calibrate", "methodId": 20}
        ]
      }
    ]
  }
]
```

| Field | Type | Purpose |
|-------|------|---------|
| `serviceTypeName` | string | Unique service type identifier |
| `serviceId` | uint16 | Numeric ID for filesystem paths |
| `eventId` | uint8 | Per-event identifier within service |
| `fieldId` | uint8 | Per-field identifier |
| `methodId` | uint8 | Per-method identifier |

---

### Section 2: serviceInstances — Deploy the Interface

```json
"serviceInstances": [
  {
    "instanceSpecifier": "my_app/temperature_port",
    "serviceTypeName": "/score/services/TemperatureService",
    "version": {"major": 1, "minor": 0},
    "instances": [
      {
        "instanceId": 1,
        "asil-level": "B",
        "binding": "SHM",
        "shm-size": 10000,
        "control-asil-b-shm-size": 20000,
        "control-qm-shm-size": 15000,
        "events": [
          {
            "eventName": "Temperature",
            "numberOfSampleSlots": 10,
            "maxSubscribers": 5,
            "numberOfIpcTracingSlots": 0
          }
        ],
        "fields": [
          {
            "fieldName": "SensorStatus",
            "numberOfSampleSlots": 4,
            "maxSubscribers": 3,
            "numberOfIpcTracingSlots": 0
          }
        ],
        "methods": [
          {
            "methodName": "Calibrate",
            "queueSize": 1,
            "enabled": true
          }
        ],
        "allowedConsumer": {
          "QM": [1000, 0],
          "B": [1001, 0]
        },
        "allowedProvider": {
          "QM": [2000, 0],
          "B": [2000, 0]
        }
      }
    ]
  }
]
```

| Field | Purpose |
|-------|---------|
| `instanceSpecifier` | Maps to your code's `InstanceSpecifier` string |
| `instanceId` | Unique deployment instance (filesystem path component) |
| `asil-level` | "B" (safety) or "QM" (quality management) |
| `shm-size` | Data shared memory size in bytes |
| `numberOfSampleSlots` | How many slots per event (ring buffer depth) |
| `maxSubscribers` | Maximum concurrent proxy subscriptions |
| `queueSize` | Method call queue depth (1 = synchronous) |
| `allowedConsumer/Provider` | UIDs allowed to connect |

---

### Section 3: global — Process Settings

```json
"global": {
  "asil-level": "B",
  "applicationID": 1234,
  "queue-size": {
    "QM-receiver": 10,
    "B-receiver": 10,
    "B-sender": 20
  },
  "shm-size-calc-mode": "SIMULATION"
}
```

| Field | Purpose |
|-------|---------|
| `asil-level` | This process's safety level |
| `applicationID` | Used for crash detection (PID mapping) |
| `queue-size` | Message passing queue pre-allocation |
| `shm-size-calc-mode` | Memory calculation strategy |

---

### Section 4: tracing — Optional IPC Tracing

```json
"tracing": {
  "enable": true,
  "applicationInstanceID": "my_app_instance",
  "traceFilterConfigPath": "./trace_filter.json"
}
```

---

## How Configuration Is Loaded

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Option 1: Command-line argument                                   │
│  ./my_app -service_instance_manifest /path/to/mw_com_config.json  │
│                                                                    │
│  Option 2: Default path (searched automatically)                   │
│  ./my_app                                                          │
│                                                                    │
│  Option 3: Explicit in code                                        │
│  score::mw::com::runtime::Initialize(argc, argv);                 │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### In Code:

```cpp
#include "score/mw/com/runtime.h"

int main(int argc, char** argv) {
    // Parses -service_instance_manifest from argv
    score::mw::com::runtime::Initialize(argc, argv);
    
    // Now InstanceSpecifiers will resolve using the loaded config
    auto spec = InstanceSpecifier::Create("my_app/temperature_port");
    // → resolves to serviceId:42, instanceId:1
}
```

> **Related files**:
> - [score/mw/com/runtime_configuration.h](../../score/mw/com/runtime_configuration.h)
> - [score/mw/com/impl/configuration/](../../score/mw/com/impl/configuration/)

---

## Sizing Guidelines

### Number of Sample Slots

```
Formula: numberOfSampleSlots ≥ 1 + Σ(max_sample_count of all subscribers)

Example:
  3 subscribers, each with Subscribe(5)
  → Need at least 1 + 15 = 16 slots

  If fewer slots: subscribers may miss samples or block
  If more slots: wastes shared memory
```

### Shared Memory Size

```
shm-size ≥ numberOfSampleSlots × sizeof(YourDataType) + overhead

Example:
  Data type: 1024 bytes
  Slots: 16
  → shm-size ≈ 16 × 1024 + headers ≈ 18000 bytes
```

---

## Permission Model

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  "allowedConsumer": {                                              │
│    "QM": [uid, secondary_gid],     ← Who can subscribe (QM)      │
│    "B": [uid, secondary_gid]       ← Who can subscribe (ASIL-B)  │
│  },                                                                │
│  "allowedProvider": {                                              │
│    "QM": [uid, secondary_gid],     ← Who can offer (QM)          │
│    "B": [uid, secondary_gid]       ← Who can offer (ASIL-B)      │
│  }                                                                 │
│                                                                    │
│  ENFORCED AT:                                                      │
│  • Service discovery (flag file permissions)                       │
│  • Shared memory creation (mmap permissions)                       │
│  • Message passing connection (SO_PEERCRED check)                 │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| [score/mw/com/impl/configuration/](../../score/mw/com/impl/configuration/) | Config parsing code |
| [score/mw/com/impl/configuration/mw_com_config_schema.json](../../score/mw/com/impl/configuration/mw_com_config_schema.json) | JSON schema |
| [score/mw/com/impl/configuration/example/](../../score/mw/com/impl/configuration/example/) | Example configs |
| [score/mw/com/example/ipc_bridge/etc/](../../score/mw/com/example/ipc_bridge/etc/) | IPC bridge config |
| [score/mw/com/runtime_configuration.h](../../score/mw/com/runtime_configuration.h) | Runtime config loading |
| [score/mw/com/runtime.h](../../score/mw/com/runtime.h) | Initialization API |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **JSON manifest** | Maps InstanceSpecifier → deployment (IDs, sizes, permissions) |
| **serviceTypes** | Abstract interface definition (events, fields, methods) |
| **serviceInstances** | Concrete deployment (buffer sizes, max subscribers, ASIL level) |
| **global** | Process-level settings (ASIL level, app ID, queue sizes) |
| **numberOfSampleSlots** | Ring buffer depth per event in shared memory |
| **maxSubscribers** | Limits concurrent proxies per event |
| **allowedConsumer/Provider** | UID-based access control |
| **-service_instance_manifest** | CLI arg to specify config path |

---

## Next: [Part 9 — Building & Testing →](part9_building_testing.md)

How to build, test, lint, and run quality checks.

---

## Previous: [← Part 7 — Safety & Transactions](part7_safety_transactions.md)
