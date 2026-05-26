# Part 10: Contributing — Your First Contribution

## Overview

This part provides a complete guide from setting up your environment to landing your first PR.

---

## Step 0: Prerequisites

```bash
# Sign the Eclipse Contributor Agreement (MANDATORY)
# https://www.eclipse.org/legal/ECA.php
# PRs from unsigned contributors will be rejected.

# Required tools:
# • Bazel 6.0+ (https://bazel.build/install)
# • Docker (https://docs.docker.com/engine/install/)
# • GCC 12+ with C++17
# • Git
```

---

## Step 1: Fork & Clone

```bash
# Fork on GitHub: https://github.com/eclipse-score/communication

# Clone your fork
git clone https://github.com/YOUR_USERNAME/communication.git
cd communication

# Add upstream remote
git remote add upstream https://github.com/eclipse-score/communication.git
```

---

## Step 2: Setup Development Environment

```bash
# Ubuntu 24.04: Unblock user namespaces for Bazel
bash actions/unblock_user_namespace_for_linux_sandbox/action_callable.sh
bazel shutdown

# Verify build works
bazel build //...

# Verify tests pass
bazel test //...
```

**Recommended**: Use the [DevContainer](https://containers.dev/) for a pre-configured environment.

---

## Step 3: Create Feature Branch

```bash
git checkout -b feature/your-feature-name
```

---

## Step 4: Make Your Changes

### Code Standards

| Rule | Detail |
|------|--------|
| **Language** | C++17 |
| **Style** | Google C++ Style Guide |
| **Naming** | `snake_case` for files, variables, functions; `PascalCase` for classes |
| **Headers** | Apache 2.0 copyright header on every file |
| **Tests** | Required for all new functionality |
| **Documentation** | Update design docs if architecture changes |

### File Organization Pattern

```
score/mw/com/impl/
├── my_feature.h              ← Header (public interface)
├── my_feature.cpp            ← Implementation
├── my_feature_test.cpp       ← Unit test
└── BUILD                     ← Add your targets here
```

### BUILD File Pattern

```python
cc_library(
    name = "my_feature",
    srcs = ["my_feature.cpp"],
    hdrs = ["my_feature.h"],
    deps = [
        "//score/mw/com/impl:existing_dep",
    ],
)

cc_unit_test(
    name = "my_feature_test",
    srcs = ["my_feature_test.cpp"],
    deps = [
        ":my_feature",
        "@googletest//:gtest_main",
    ],
)
```

---

## Step 5: Verify Your Changes

```bash
# Build
bazel build //...

# Run all tests
bazel test //...

# Format code
bazel run //:format

# Check copyright headers
bazel run //:copyright.check

# Run sanitizers (optional but recommended)
bazel test --config=asan_ubsan_lsan //...
```

---

## Step 6: Commit & Push

```bash
git add .
git commit -m "feat: Add description of your change

Detailed explanation of what and why.
Reference any related issues: #123"

git push origin feature/your-feature-name
```

---

## Step 7: Open Pull Request

1. Go to your fork on GitHub
2. Click "New Pull Request"
3. Base: `eclipse-score/communication:main` ← Head: `your-fork:feature/your-feature-name`
4. Write clear title and description
5. Reference related issues
6. Wait for CI and review

---

## Where to Start (Good First Contributions)

```
┌────────────────────────────────────────────────────────────────────────┐
│                                                                        │
│  DIFFICULTY: EASY                                                      │
│  ─────────────────                                                     │
│  • Add missing unit tests for existing code                           │
│  • Fix documentation typos or add examples                            │
│  • Improve error messages                                              │
│  • Add const-correctness where missing                                │
│                                                                        │
│  DIFFICULTY: MEDIUM                                                    │
│  ─────────────────────                                                 │
│  • Add new test cases for edge cases                                  │
│  • Improve performance benchmark coverage                             │
│  • Add tracing instrumentation                                        │
│  • Fix clang-tidy warnings                                            │
│                                                                        │
│  DIFFICULTY: ADVANCED                                                  │
│  ─────────────────────────                                             │
│  • Implement new features (check GitHub Issues)                       │
│  • Add SOME/IP binding for inter-ECU communication                    │
│  • Performance optimization in lock-free paths                        │
│  • Extend Rust API coverage                                           │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Understanding the Test Pattern

Most LoLa tests follow this pattern:

```cpp
#include "score/mw/com/impl/my_feature.h"

#include <gtest/gtest.h>
#include <gmock/gmock.h>

namespace score::mw::com::impl
{
namespace
{

class MyFeatureTest : public ::testing::Test
{
  protected:
    void SetUp() override {
        // Setup test fixtures
    }
    
    void TearDown() override {
        // Cleanup
    }
    
    // Test helpers and mocks
    MockDependency mock_dep_;
};

TEST_F(MyFeatureTest, DescriptiveTestName) {
    // Given
    auto sut = MyFeature(mock_dep_);
    
    // When
    auto result = sut.DoSomething();
    
    // Then
    EXPECT_TRUE(result.has_value());
    EXPECT_EQ(result.value(), expected);
}

}  // namespace
}  // namespace score::mw::com::impl
```

---

## Community & Support

| Resource | Link |
|----------|------|
| **Issues** | https://github.com/eclipse-score/communication/issues |
| **Discussions** | https://github.com/eclipse-score/communication/discussions |
| **ECA Signing** | https://www.eclipse.org/legal/ECA.php |
| **Eclipse Score** | https://eclipse-score.github.io/score/ |

---

## Contribution Checklist

- [ ] Eclipse Contributor Agreement signed
- [ ] Feature branch created from latest `main`
- [ ] Code follows C++17 / Google Style Guide
- [ ] Copyright headers on all new files
- [ ] Unit tests added/updated
- [ ] `bazel build //...` passes
- [ ] `bazel test //...` passes
- [ ] `bazel run //:format.check` passes
- [ ] `bazel run //:copyright.check` passes
- [ ] PR description is clear with issue reference
- [ ] Design docs updated (if architecture changed)

---

## Summary

| Step | Action |
|------|--------|
| 1 | Sign ECA |
| 2 | Fork & clone |
| 3 | Setup environment (DevContainer or manual) |
| 4 | Create feature branch |
| 5 | Write code + tests (C++17, Google Style) |
| 6 | Format + lint + test |
| 7 | Push + open PR |
| 8 | Address review feedback |

---

## Previous: [← Part 9 — Building & Testing](part9_building_testing.md)

---

## Back to: [Deep Dive Index →](README.md)
