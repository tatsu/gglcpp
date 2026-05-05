---
name: google-cpp-review
description: C++ code review for memory safety, modern C++ idioms, concurrency, and Google Style compliance. Run after writing or modifying C++ code.
origin: gglcpp
---

# C++ Code Review (Google Style)

Comprehensive code review for C++ following Google conventions. Checks memory safety, modern C++ idioms, concurrency correctness, and Google Style Guide compliance.

## When to Use

- After writing or modifying C++ code
- Before committing C++ changes
- Reviewing pull requests with C++ code
- Checking for memory safety or concurrency issues

### When NOT to Use

- Non-C++ files
- Tasks unrelated to code quality (build failures → use `gglcpp:google-cpp-build`)

## Diagnostic Commands

```bash
# Identify changed C++ files
git diff --name-only -- '*.cc' '*.h' '*.cpp' '*.hpp'

# Google style lint
cpplint --recursive src/

# Clang-tidy with Google-relevant checks
clang-tidy src/**/*.cc -- -std=c++20

# Additional analysis
cppcheck --enable=all --suppress=missingIncludeSystem src/

# Build with full warnings
cmake --build build -- -Wall -Wextra -Wpedantic
```

---

## Review Priorities

### CRITICAL — Memory Safety

- **Raw `new`/`delete`**: Use `std::unique_ptr` or `std::shared_ptr` instead.
- **Buffer overflows**: C-style arrays, `strcpy`, `sprintf` without bounds checks.
- **Use-after-free**: Dangling pointers, invalidated iterators.
- **Uninitialized variables**: Reading before assignment.
- **Memory leaks**: Missing RAII; resources not tied to object lifetime.
- **Null dereference**: Pointer access without null check.

```cpp
// WRONG
MyClass* obj = new MyClass();
// ...
delete obj;  // easy to miss, double-delete risk

// CORRECT (Google Style)
auto obj = std::make_unique<MyClass>();
// automatic cleanup
```

### CRITICAL — Security

- **Command injection**: Unvalidated input passed to `system()` or `popen()`.
- **Format string attacks**: User input used as `printf` format string.
- **Integer overflow**: Unchecked arithmetic on untrusted input.
- **Hardcoded secrets**: API keys, passwords in source code.
- **Unsafe casts**: `reinterpret_cast` without justification.

### HIGH — Concurrency

- **Data races**: Shared mutable state without synchronization.
- **Deadlocks**: Multiple mutexes locked in inconsistent order.
- **Missing lock guards**: Manual `lock()`/`unlock()` instead of `absl::MutexLock` or `std::lock_guard`.
- **Detached threads**: `std::thread` without `join()` or `detach()`.

```cpp
// WRONG
mu_.Lock();
DoWork();
mu_.Unlock();  // skipped if DoWork() throws or returns early

// CORRECT (Google / Abseil style)
absl::MutexLock lock(&mu_);
DoWork();
```

### HIGH — Code Quality

- **No RAII**: Manual resource management.
- **Rule of Five violations**: Incomplete special member functions when one is user-defined.
- **Large functions**: Over 50 lines — split into focused helpers.
- **Deep nesting**: More than 4 levels — use early returns.
- **C-style code**: `malloc`, C arrays, `typedef` instead of `using`.
- **No exceptions**: Google Style prohibits exceptions; use `absl::Status`/`absl::StatusOr` for error propagation.

```cpp
// WRONG (exceptions)
MyClass::MyClass() {
  if (!Init()) throw std::runtime_error("init failed");
}

// CORRECT (Google Style)
absl::Status MyClass::Init() {
  if (!DoInit()) return absl::InternalError("init failed");
  return absl::OkStatus();
}
```

### MEDIUM — Performance

- **Unnecessary copies**: Pass large objects by `const&` instead of by value.
- **Missing move semantics**: Not using `std::move` for sink parameters.
- **String concatenation in loops**: Use `absl::StrCat` or `std::string::reserve`.
- **Missing `reserve()`**: Known-size `std::vector` without pre-allocation.

```cpp
// WRONG
void Process(std::string data) { ... }  // copies on every call

// CORRECT
void Process(std::string_view data) { ... }     // read-only
void Process(std::string data) { ... }           // sink (will be stored/moved)
void Process(const std::string& data) { ... }    // read-only, owned string
```

### MEDIUM — Google Style Compliance

- **Naming**: functions and methods `PascalCase`, variables `snake_case`, constants `kConstantName`, macros `MY_MACRO`.
- **`using namespace`**: Never in headers; avoid even in `.cc` files.
- **Include guards**: `#ifndef PROJECT_PATH_FILE_H_` format, not `#pragma once`.
- **Include order**: own header → C system → C++ stdlib → third-party → project (see below).
- **`const` correctness**: Mark methods `const` when they don't mutate state.
- **`[[nodiscard]]`**: Apply to functions returning `absl::Status` or `absl::StatusOr`.
- **Line length**: 80 characters maximum.
- **Trailing whitespace / formatting**: Run `clang-format --style=Google`.

```cpp
// WRONG naming
class myClass {                    // should be PascalCase
  int MyVariable;                  // should be snake_case
  void my_method() const;          // should be PascalCase
  static const int MAX_SIZE = 10;  // should be kMaxSize
};

// CORRECT (Google Style)
class MyClass {
  int my_variable_;                // trailing underscore for members
  void MyMethod() const;
  static constexpr int kMaxSize = 10;
};
```

### MEDIUM — Abseil Preferences

Google codebases use Abseil. Prefer Abseil over rolling equivalents:

| Instead of | Prefer |
|-----------|--------|
| `std::unordered_map` | `absl::flat_hash_map` |
| `std::unordered_set` | `absl::flat_hash_set` |
| Manual string split | `absl::StrSplit` |
| Manual string join | `absl::StrJoin` |
| Manual string format | `absl::StrFormat` or `absl::StrCat` |
| `std::this_thread::sleep_for` | `absl::SleepFor` |
| Exception-based errors | `absl::Status` / `absl::StatusOr` |

---

## Google Style: Include Order

```cpp
// 1. This file's own header (for .cc files)
#include "my_module/my_class.h"

// 2. C system headers
#include <sys/types.h>

// 3. C++ standard library headers
#include <memory>
#include <string>

// 4. Third-party library headers
#include "absl/status/statusor.h"
#include "absl/strings/string_view.h"

// 5. Project headers
#include "other_module/util.h"
```

Each group separated by a blank line; within each group, alphabetical order.

---

## Approval Criteria

| Status | Condition |
|--------|-----------|
| **Approve** | No CRITICAL or HIGH issues |
| **Warning** | MEDIUM issues only — merge with caution |
| **Block** | Any CRITICAL or HIGH issue found |

---

## Output Format

```
[CRITICAL] Memory leak
File: src/my_module/my_service.cc:45
Issue: Raw `new` without matching `delete` or RAII wrapper.
Fix: Replace with `std::make_unique<Session>(user_id)`.

[HIGH] Missing lock guard
File: src/my_module/my_service.cc:78
Issue: Manual mutex lock/unlock — skipped on early return.
Fix: Use `absl::MutexLock lock(&mu_);`.

Summary:
  CRITICAL: 1
  HIGH:     1
  MEDIUM:   0
Recommendation: BLOCK — fix CRITICAL and HIGH before merging.
```

---

## Related Skills

- `gglcpp:google-cpp-build` — fix build errors before reviewing
- `gglcpp:google-cpp-testing` — test coverage standards and patterns
- `gglcpp:google-cpp-style-guide` — authoritative style reference
