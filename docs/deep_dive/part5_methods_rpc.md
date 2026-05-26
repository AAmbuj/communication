# Part 5: Methods (RPC) — Request/Response Communication

## Overview

While Events are one-way (fire-and-forget), **Methods** provide **request/response** (RPC) semantics. A proxy calls a method on a skeleton, passes arguments, and receives a return value — all through shared memory with zero-copy support.

---

## Methods vs Events

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  EVENT (Parts 3-4)                    METHOD (This Part)               │
│  ════════════════                     ══════════════════               │
│                                                                        │
│  Skeleton ──────► Proxy               Proxy ──────► Skeleton           │
│  (one-way, push)                      (request)                        │
│                                              │                         │
│                                       Skeleton ──────► Proxy           │
│                                       (response)                       │
│                                                                        │
│  "Here's the latest sensor data"      "Calculate route from A to B"   │
│  "Temperature is 42.5°C"             "Result: [route data]"           │
│                                                                        │
│  Asynchronous (fire & forget)         Synchronous (call & wait)       │
│  1:N (one sender, many receivers)     1:1 (one caller, one handler)   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Class Hierarchy

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                                                                             │
│  PROXY SIDE (Caller)                   SKELETON SIDE (Handler)              │
│                                                                             │
│  ┌─────────────────────────┐           ┌─────────────────────────┐         │
│  │ ProxyMethod<R(Args...)> │           │ SkeletonMethod<R(Args)> │         │
│  │                         │           │                         │         │
│  │ • operator()(args...)   │           │ • RegisterHandler(fn)   │         │
│  │ • Allocate()            │           │                         │         │
│  └───────────┬─────────────┘           └───────────┬─────────────┘         │
│              │ inherits                            │ inherits              │
│  ┌───────────▼─────────────┐           ┌───────────▼─────────────┐         │
│  │ ProxyMethodBase         │           │ SkeletonMethodBase      │         │
│  └───────────┬─────────────┘           └───────────┬─────────────┘         │
│              │ delegates                           │ delegates             │
│  ┌───────────▼─────────────┐           ┌───────────▼─────────────┐         │
│  │ ProxyMethodBinding      │           │ SkeletonMethodBinding   │         │
│  │ (interface)             │           │ (interface)             │         │
│  └───────────┬─────────────┘           └───────────┬─────────────┘         │
│              │ implemented by                      │ implemented by        │
│  ┌───────────▼─────────────┐           ┌───────────▼─────────────┐         │
│  │ lola::ProxyMethod       │           │ lola::SkeletonMethod    │         │
│  │ (shared memory impl)    │           │ (shared memory impl)    │         │
│  └─────────────────────────┘           └─────────────────────────┘         │
│                                                                             │
│  TEMPLATE SPECIALIZATIONS:                                                  │
│  ┌──────────────────────────────────────────────────────────────┐          │
│  │ ProxyMethod<ReturnType(ArgTypes...)>  — full signature        │          │
│  │ ProxyMethod<void(ArgTypes...)>        — no return value       │          │
│  │ ProxyMethod<ReturnType()>             — no arguments          │          │
│  │ ProxyMethod<void()>                   — fire and forget       │          │
│  └──────────────────────────────────────────────────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## The Request-Response Cycle

```
   PROXY PROCESS                                    SKELETON PROCESS
   ═════════════                                    ════════════════

   1. proxy_method(arg1, arg2)
         │
         ▼
   ┌─────────────────────────┐
   │ Allocate in-args buffer │
   │ in shared memory        │
   │ (TypeErasedCallQueue)   │
   └───────────┬─────────────┘
               │
         ▼
   ┌─────────────────────────┐
   │ Serialize args into     │
   │ shared memory buffer    │         ┌──────────────────────────┐
   │ (zero-copy if using     │         │                          │
   │  Allocate() variant)    │         │   SHARED MEMORY          │
   └───────────┬─────────────┘         │                          │
               │                        │  ┌────────────────────┐ │
               ├───── write args ──────►│  │ In-Args Buffer     │ │
               │                        │  │ [arg1][arg2]       │ │
               │                        │  ├────────────────────┤ │
               │                        │  │ Return Value Buffer│ │
               │                        │  │ [     empty      ] │ │
               │                        │  └────────────────────┘ │
               │                        │                          │
   ┌───────────▼─────────────┐         └──────────────────────────┘
   │ DoCall() →              │                     │
   │ Send notification via   │                     │
   │ message passing:        │─── "method call!" ──┤
   │ "args ready in slot N"  │                     │
   └───────────┬─────────────┘                     │
               │                                    ▼
               │ (blocks/waits)        ┌─────────────────────────┐
               │                       │ Notification received    │
               │                       │                         │
               │                       │ 1. Deserialize args     │
               │                       │    from shared memory   │
               │                       │                         │
               │                       │ 2. Call user handler:   │
               │                       │    handler(arg1, arg2)  │
               │                       │                         │
               │                       │ 3. Write return value   │
               │                       │    to return buffer     │
               │                       └──────────┬──────────────┘
               │                                   │
               │                                   │ write return
               │         ┌──────────────────────┐  │
               │         │  Return Value Buffer │◄─┘
               │         │  [result_data]       │
               │         └──────────┬───────────┘
               │                    │
               │◄── read return ────┘
               │
   ┌───────────▼─────────────┐
   │ Return MethodReturnType │
   │ Ptr to caller           │
   │ (points to SHM)        │
   └─────────────────────────┘
         │
         ▼
   auto result = *return_ptr;  // Zero-copy read!
```

---

## Two Ways to Call Methods

### Way 1: Copying (Simple)

```cpp
// Arguments are copied into shared memory internally
auto result = proxy.calculate_route_(start_point, end_point);

if (result.has_value()) {
    auto& route = *result.value();  // Direct pointer to return value in SHM
    display(route);
}
```

**Internally**: `Allocate()` + copy args + `DoCall()`

### Way 2: Zero-Copy (Efficient for large args)

```cpp
// Step 1: Allocate argument buffers in shared memory
auto alloc_result = proxy.calculate_route_.Allocate();
auto [start_ptr, end_ptr] = std::move(alloc_result.value());

// Step 2: Write directly to shared memory (no copy!)
*start_ptr = StartPoint{48.1, 11.5};
*end_ptr = EndPoint{52.5, 13.4};

// Step 3: Call with pre-filled args
auto result = proxy.calculate_route_(std::move(start_ptr), std::move(end_ptr));

// Step 4: Read result (also zero-copy from SHM)
auto& route = *result.value();
```

---

## Skeleton Handler Registration

```cpp
class MySkeletonImpl : public MySkeleton {
  public:
    MySkeletonImpl(InstanceSpecifier spec) : MySkeleton(spec) {
        // Register handler for incoming method calls
        calculate_route_.RegisterHandler(
            [this](const StartPoint& start, const EndPoint& end) -> RouteResult {
                // This runs in the skeleton's handler thread
                // when a proxy calls this method
                return compute_route(start, end);
            }
        );
    }
};
```

---

## Shared Memory Layout for Methods

```
┌─────────────────────────────────────────────────────────────────────┐
│                  TypeErasedCallQueue                                  │
│                                                                     │
│  Per-method, per-proxy instance:                                    │
│                                                                     │
│  Queue Position 0 (currently only 1 slot = synchronous):           │
│  ┌──────────────────────────────────────────────────────────┐      │
│  │                                                          │      │
│  │  ┌─────────────────────┐  ┌─────────────────────────┐  │      │
│  │  │  In-Args Buffer     │  │  Return Value Buffer    │  │      │
│  │  │                     │  │                         │  │      │
│  │  │  Size: sum of       │  │  Size: sizeof(Return)   │  │      │
│  │  │  sizeof(each arg)   │  │  + alignment            │  │      │
│  │  │  + alignment        │  │                         │  │      │
│  │  │                     │  │                         │  │      │
│  │  │  Layout:            │  │  Written by: skeleton   │  │      │
│  │  │  [Arg1|pad|Arg2|..] │  │  Read by: proxy        │  │      │
│  │  │                     │  │                         │  │      │
│  │  │  Written by: proxy  │  │                         │  │      │
│  │  │  Read by: skeleton  │  │                         │  │      │
│  │  └─────────────────────┘  └─────────────────────────┘  │      │
│  │                                                          │      │
│  └──────────────────────────────────────────────────────────┘      │
│                                                                     │
│  NOTE: Queue size is currently 1 (synchronous only).               │
│  Infrastructure supports future async (queue_size > 1).            │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Smart Pointer Types for Methods

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  MethodInArgPtr<T>                  MethodReturnTypePtr<T>         │
│  ═════════════════                  ═════════════════════          │
│                                                                    │
│  • Points to in-arg buffer         • Points to return buffer      │
│    in shared memory                  in shared memory              │
│  • Writable by proxy               • Read-only for proxy          │
│  • Tracks queue_position            • Tracks queue_position       │
│  • Move-only (no copy)             • Move-only (no copy)          │
│  • Active flag prevents            • Active flag prevents         │
│    slot reuse                        slot reuse                   │
│                                                                    │
│  Usage:                             Usage:                         │
│  *arg_ptr = my_value;               auto& val = *return_ptr;     │
│  method(std::move(arg_ptr));         use(val.field);              │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/methods/method_signature_element_ptr.h](../../score/mw/com/impl/methods/method_signature_element_ptr.h)

---

## Type Erasure — How Types Cross the Binding Boundary

The binding layer doesn't know about your custom types. It only sees **raw bytes**:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  API Layer (knows types)          Binding Layer (type-erased)   │
│                                                                 │
│  ProxyMethod<Route(Start, End)>   ProxyMethodBinding            │
│       │                                │                        │
│       │  Serialize<Start, End>()       │  GetInArgsBuffer()     │
│       │  into byte buffer    ─────────►│  → span<std::byte>     │
│       │                                │                        │
│       │  DoCall() ────────────────────►│  DoCall(queue_pos)     │
│       │                                │  → sends notification  │
│       │                                │                        │
│       │  Deserialize<Route>() ◄────────│  GetReturnBuffer()     │
│       │  from byte buffer              │  → span<std::byte>     │
│       │                                │                        │
│                                                                 │
│  compile-time type info:                                        │
│  CreateDataTypeSizeInfoFromTypes<Start, End>()                  │
│  → { total_size, alignment, offsets[] }                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> **Related file**: [score/mw/com/impl/util/type_erased_storage.h](../../score/mw/com/impl/util/type_erased_storage.h)

---

## Method Identification

Each method is uniquely identified within a service:

```
UniqueMethodIdentifier = {
    element_id: "calculate_route"    // method name
    method_type: MethodType::kMethod // vs kFieldGetter, kFieldSetter
}

ProxyMethodInstanceIdentifier = {
    proxy_identifier: <which proxy instance>
    unique_method_id: <which method on that proxy>
}
```

> **Related files**:
> - [score/mw/com/impl/bindings/lola/methods/unique_method_identifier.h](../../score/mw/com/impl/bindings/lola/methods/unique_method_identifier.h)
> - [score/mw/com/impl/bindings/lola/methods/proxy_method_instance_identifier.h](../../score/mw/com/impl/bindings/lola/methods/proxy_method_instance_identifier.h)

---

## Methods in Fields (Get/Set)

Fields internally use methods for their Get() and Set() operations:

```
┌────────────────────────────────────────────────────────────────┐
│                                                                │
│  ProxyField<T>                                                 │
│                                                                │
│  ┌────────────────────────────────┐                            │
│  │ Get() → ProxyMethod<T()>      │  (no args, returns T)      │
│  │         MethodType::kFieldGetter                            │
│  ├────────────────────────────────┤                            │
│  │ Set(value) → ProxyMethod<void(T)>  (arg=T, no return)     │
│  │              MethodType::kFieldSetter                       │
│  ├────────────────────────────────┤                            │
│  │ Subscribe() → Event mechanism  │  (notified on change)     │
│  └────────────────────────────────┘                            │
│                                                                │
│  A Field = Event (notify) + Method (get) + Method (set)        │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

## Current Limitations & Future

| Aspect | Current | Future |
|--------|---------|--------|
| Call queue size | 1 (synchronous only) | N (async support) |
| Concurrency | One call at a time per method | Concurrent calls |
| Timeout | Not implemented | Configurable timeout |
| Error handling | Basic Result<> | Rich error codes |

---

## Key Source Files

| File | Purpose |
|------|---------|
| [score/mw/com/impl/methods/](../../score/mw/com/impl/methods/) | Binding-independent method classes |
| [proxy_method_base.h](../../score/mw/com/impl/methods/proxy_method_base.h) | Base class for proxy methods |
| [skeleton_method_base.h](../../score/mw/com/impl/methods/skeleton_method_base.h) | Base class for skeleton methods |
| [proxy_method_binding.h](../../score/mw/com/impl/methods/proxy_method_binding.h) | Binding interface (proxy side) |
| [skeleton_method_binding.h](../../score/mw/com/impl/methods/skeleton_method_binding.h) | Binding interface (skeleton side) |
| [method_signature_element_ptr.h](../../score/mw/com/impl/methods/method_signature_element_ptr.h) | Smart pointer types |
| [score/mw/com/impl/bindings/lola/methods/](../../score/mw/com/impl/bindings/lola/methods/) | LoLa binding implementation |
| [type_erased_call_queue.h](../../score/mw/com/impl/bindings/lola/methods/type_erased_call_queue.h) | SHM storage for call data |
| [unique_method_identifier.h](../../score/mw/com/impl/bindings/lola/methods/unique_method_identifier.h) | Method ID |
| [score/mw/com/design/methods/](../../score/mw/com/design/methods/) | Design documentation |

---

## Summary

| Concept | One-liner |
|---------|-----------|
| **ProxyMethod** | Caller side — allocates args, invokes call, receives return |
| **SkeletonMethod** | Handler side — registers callback, processes requests |
| **TypeErasedCallQueue** | Pre-allocated SHM buffer for args + return per method |
| **MethodInArgPtr** | Zero-copy pointer to argument buffer in SHM |
| **MethodReturnTypePtr** | Zero-copy pointer to return value in SHM |
| **Queue size = 1** | Currently synchronous only (one call at a time) |
| **Type erasure** | API knows types; binding sees raw byte buffers |

---

## Next: [Part 6 — Message Passing Layer →](part6_message_passing.md)

How do notifications travel between processes? What are Unix domain sockets and QNX dispatch?

---

## Previous: [← Part 4 — Event System](part4_event_system.md)
