# Part 9: Building & Testing — Bazel, Tests, and Quality Checks

## Overview

LoLa uses **Bazel** as its build system. This part covers how to build, run tests, check code quality, and understand the test infrastructure.

---

## Quick Reference Commands

```bash
# ═══════ BUILD ═══════
bazel build //...                          # Build everything
bazel build //score/mw/com/impl:all        # Build mw::com implementation
bazel build //score/message_passing:all    # Build message passing

# ═══════ TEST ═══════
bazel test //...                           # Run ALL tests
bazel test //score/mw/com/impl:all         # Test mw::com impl
bazel test //score/mw/com/impl/bindings/lola:all  # Test LoLa binding
bazel test //score/message_passing:all     # Test message passing

# ═══════ LINT & FORMAT ═══════
bazel run //:format.check                  # Check formatting (no changes)
bazel run //:format                        # Auto-fix formatting
bazel run //:copyright.check               # Check copyright headers
bazel run //:copyright.fix                 # Fix copyright headers

# ═══════ STATIC ANALYSIS ═══════
bazel test --config=clang-tidy //...       # AUTOSAR compliance check

# ═══════ SANITIZERS ═══════
bazel test --config=asan_ubsan_lsan //...  # Address + UB + Leak sanitizer
bazel test --config=tsan //...             # Thread sanitizer

# ═══════ COVERAGE ═══════
bazel coverage //...                       # Generate coverage data
```

---

## Build System Layout

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  MODULE.bazel          ← Root module definition + dependencies          │
│  BUILD                 ← Root targets (format, copyright, etc.)         │
│  WORKSPACE             ← Legacy workspace file                          │
│                                                                         │
│  score/                                                                 │
│  ├── mw/com/                                                           │
│  │   ├── BUILD         ← mw::com library targets                       │
│  │   ├── impl/                                                         │
│  │   │   ├── BUILD     ← impl library + unit tests                    │
│  │   │   └── bindings/lola/                                           │
│  │   │       └── BUILD ← LoLa binding library + tests                 │
│  │   ├── example/      ← Example applications                          │
│  │   └── test/         ← Integration tests                            │
│  └── message_passing/                                                  │
│      └── BUILD         ← Message passing library + tests               │
│                                                                         │
│  quality/                                                              │
│  ├── unit_testing/     ← cc_unit_test() macro                         │
│  ├── integration_testing/ ← integration_test() macro                  │
│  ├── sanitizer/        ← ASAN/TSAN/UBSAN configs                      │
│  ├── static_analysis/  ← Clang-tidy + CodeQL                          │
│  └── coverage/         ← LLVM coverage setup                          │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Test Types

### 1. Unit Tests (GoogleTest)

```
┌────────────────────────────────────────────────────────────────────┐
│  Pattern: cc_unit_test(name = "foo_test", srcs = ["foo_test.cpp"]) │
│                                                                    │
│  • Uses GoogleTest framework                                       │
│  • Built for BOTH Linux and QNX targets                           │
│  • Located alongside source: foo.h, foo.cpp, foo_test.cpp         │
│  • Run: bazel test //path/to:foo_test                             │
│                                                                    │
│  Example:                                                          │
│  bazel test //score/mw/com/impl:instance_specifier_test            │
│  bazel test //score/mw/com/impl/bindings/lola:event_slot_status_test│
└────────────────────────────────────────────────────────────────────┘
```

### 2. Integration Tests (Container-based)

```
┌────────────────────────────────────────────────────────────────────┐
│  Pattern: integration_test(name, srcs, filesystem, ...)            │
│                                                                    │
│  • Run in OCI containers (Ubuntu 24.04)                           │
│  • Or QNX8 QEMU environments                                      │
│  • Test cross-process communication                               │
│  • Verify real shared memory + message passing                    │
│                                                                    │
│  Run: bazel test //score/mw/com/test:integration_tests             │
└────────────────────────────────────────────────────────────────────┘
```

### 3. Sanitizer Tests

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  ASAN (Address Sanitizer):                                         │
│    Detects: buffer overflow, use-after-free, double-free           │
│    Run: bazel test --config=asan //...                             │
│                                                                    │
│  UBSAN (Undefined Behavior Sanitizer):                             │
│    Detects: signed overflow, null deref, alignment issues          │
│    Run: bazel test --config=ubsan //...                            │
│                                                                    │
│  LSAN (Leak Sanitizer):                                            │
│    Detects: memory leaks                                           │
│    Run: bazel test --config=lsan //...                             │
│                                                                    │
│  TSAN (Thread Sanitizer):                                          │
│    Detects: data races, deadlocks                                  │
│    Run: bazel test --config=tsan //...                             │
│                                                                    │
│  Combined:                                                         │
│    bazel test --config=asan_ubsan_lsan //...                       │
│                                                                    │
│  Suppression files: quality/sanitizer/*.supp                       │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

### 4. Static Analysis

```
┌────────────────────────────────────────────────────────────────────┐
│                                                                    │
│  Clang-Tidy (AUTOSAR C++14 compliance):                           │
│    bazel test --config=clang-tidy //...                            │
│    Config: .clang-tidy in project root                             │
│                                                                    │
│  CodeQL (MISRA C++ checking):                                      │
│    bazel run //quality/static_analysis:codeql_lint -- --target=//..│
│    Output: codeql.sarif, codeql.csv                                │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
```

---

## Prerequisites Setup (Ubuntu 24.04)

```bash
# 1. Unblock user namespaces for Bazel sandbox
bash actions/unblock_user_namespace_for_linux_sandbox/action_callable.sh
bazel shutdown

# 2. Build and verify
bazel build //...
bazel test //...
```

---

## Running Specific Tests

```bash
# Single test
bazel test //score/mw/com/impl:proxy_event_test

# All tests in a package
bazel test //score/mw/com/impl/bindings/lola:all

# Tests matching a pattern
bazel test //score/mw/com/impl/bindings/lola/...

# With verbose output
bazel test --test_output=all //score/mw/com/impl:proxy_event_test

# Filter specific test case
bazel test --test_filter="ProxyEventTest.Subscribe" //score/mw/com/impl:proxy_event_test
```

---

## Coverage

```bash
# Generate coverage for all tests
bazel coverage //...

# Generate for specific package
bazel coverage //score/mw/com/impl:all

# Generate HTML report
genhtml bazel-out/_coverage/_coverage_report.dat --output-directory coverage_html

# Or use the project script:
bash quality/coverage/generate_coverage_html.sh
```

---

## Key Source Files

| File | Purpose |
|------|---------|
| [MODULE.bazel](../../MODULE.bazel) | Root module + dependencies |
| [BUILD](../../BUILD) | Root targets (format, copyright) |
| [quality/unit_testing/unit_testing.bzl](../../quality/unit_testing/unit_testing.bzl) | Unit test macro |
| [quality/integration_testing/integration_testing.bzl](../../quality/integration_testing/integration_testing.bzl) | Integration test macro |
| [quality/sanitizer/sanitizer.bazelrc](../../quality/sanitizer/sanitizer.bazelrc) | Sanitizer configs |
| [quality/static_analysis/](../../quality/static_analysis/) | Static analysis tools |
| [quality/coverage/](../../quality/coverage/) | Coverage scripts |
| [quality/quality.md](../../quality/quality.md) | Full quality documentation |

---

## Summary

| Task | Command |
|------|---------|
| Build all | `bazel build //...` |
| Test all | `bazel test //...` |
| Format check | `bazel run //:format.check` |
| Format fix | `bazel run //:format` |
| Copyright check | `bazel run //:copyright.check` |
| ASAN/UBSAN/LSAN | `bazel test --config=asan_ubsan_lsan //...` |
| TSAN | `bazel test --config=tsan //...` |
| Clang-tidy | `bazel test --config=clang-tidy //...` |
| Coverage | `bazel coverage //...` |

---

## Next: [Part 10 — Contributing →](part10_contributing.md)

The complete workflow for making your first contribution.

---

## Previous: [← Part 8 — Configuration & Deployment](part8_configuration.md)
