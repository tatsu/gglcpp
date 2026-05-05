---
name: google-cpp-build
description: Fix C++ build errors, CMake issues, and linker problems incrementally following Google C++ Style. Surgical fixes only — no refactoring.
origin: gglcpp
---

# C++ Build and Fix (Google Style)

Incremental build error resolution for C++ codebases following Google conventions. Fixes compilation errors, CMake issues, and linker problems with minimal, surgical changes.

## When to Use

- `cmake --build` fails with compilation or linker errors
- Template instantiation failures
- Include/dependency issues
- After pulling changes that break the build
- `cpplint` or `clang-tidy` reports blocking errors

### When NOT to Use

- Refactoring or style cleanup unrelated to build failures
- Architectural redesigns
- Missing external dependencies that require installation

## Diagnostic Workflow

Run diagnostics in this order; stop and fix at the first blocking stage:

```bash
# 1. Configure
cmake -S . -B build -DCMAKE_BUILD_TYPE=Debug 2>&1 | tail -30

# 2. Build (Google uses .cc; adjust glob if project uses .cpp)
cmake --build build 2>&1 | head -100

# 3. Google style check (cpplint)
cpplint --recursive src/ 2>&1 | grep -E "error:|warning:" | head -40

# 4. Clang-tidy with Google checks
clang-tidy src/**/*.cc -- -std=c++20 2>&1 | head -60

# 5. Tests (only after build succeeds)
ctest --test-dir build --output-on-failure
```

## Resolution Workflow

```
1. cmake --build build    → parse error, identify file:line
2. Read affected file     → understand context
3. Apply minimal fix      → only what's needed to unblock the build
4. cmake --build build    → verify fix
5. ctest --test-dir build → ensure nothing broke
```

## Common Fix Patterns

| Error | Cause | Fix |
|-------|-------|-----|
| `undefined reference to X` | Missing implementation or library | Add source file or `target_link_libraries` |
| `no matching function for call` | Wrong argument types | Fix types or add overload |
| `use of undeclared identifier` | Missing `#include` or typo | Add include or fix name |
| `multiple definition of` | Duplicate symbol | Add `inline`, move to `.cc`, or fix include guard |
| `cannot convert X to Y` | Type mismatch | Add appropriate cast |
| `incomplete type` | Forward decl used where full type needed | Replace with `#include` |
| `template argument deduction failed` | Wrong template args | Fix template parameters |
| `CMake Error` | Configuration issue | Fix `CMakeLists.txt` |
| `cpplint: build/include_order` | Includes not in Google order | Reorder: related `.h`, `<system>`, then project headers |
| `cpplint: build/namespaces` | `using namespace` in header | Remove or move to `.cc` |

## CMake Troubleshooting

```bash
cmake -S . -B build -DCMAKE_VERBOSE_MAKEFILE=ON
cmake --build build --verbose
cmake --build build --clean-first
```

## Google Style: Include Order

Google requires includes in this order within each group (alphabetical), separated by blank lines:

```cpp
// 1. Related .h (the file's own header first)
#include "my_module/my_class.h"

// 2. C system headers
#include <sys/types.h>

// 3. C++ standard library headers
#include <memory>
#include <string>
#include <vector>

// 4. Third-party library headers
#include "absl/status/status.h"
#include "absl/strings/string_view.h"

// 5. Project headers
#include "other_module/util.h"
```

Fix `cpplint build/include_order` by reordering to match this pattern.

## Google Style: Include Guards

Google uses `#ifndef` guards, not `#pragma once`. Format:
`<PROJECT>_<PATH>_<FILE>_H_`

```cpp
#ifndef MYPROJECT_SRC_FOO_BAR_H_
#define MYPROJECT_SRC_FOO_BAR_H_

// ...

#endif  // MYPROJECT_SRC_FOO_BAR_H_
```

## CMakeLists.txt Template (Google Style)

```cmake
cmake_minimum_required(VERSION 3.20)
project(example LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # needed by clang-tidy

add_library(my_lib
  src/my_class.cc
)
target_include_directories(my_lib PUBLIC include)
target_compile_options(my_lib PRIVATE -Wall -Wextra -Wpedantic)
```

## Key Principles

- **Surgical fixes only** — don't refactor, just fix the error.
- **Never** suppress warnings with `#pragma` without approval.
- **Never** change function signatures unless strictly necessary.
- Fix root cause over suppressing symptoms.
- One fix at a time; verify the build after each.

## Stop Conditions

Stop and report if:
- Same error persists after 3 fix attempts.
- A fix introduces more errors than it resolves.
- The error requires architectural changes beyond scope.
- Missing external dependencies need installation.

## Output Format

```
[FIXED] src/my_module/my_class.cc:42
Error: undefined reference to `MyService::Create`
Fix: Added missing method implementation in my_service.cc
Remaining errors: 2
```

Final line: `Build Status: SUCCESS/FAILED | Errors Fixed: N | Files Modified: list`

## Related Skills

- `gglcpp:google-cpp-testing` — run after build succeeds
- `gglcpp:google-cpp-review` — review code quality after build is green
- `gglcpp:google-cpp-style-guide` — authoritative style reference
