---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
---
# C++ Hooks (Google Style)

Automated quality checks for Google-style C++ code using cpplint, clang-format, and clang-tidy.

## PostToolUse Hooks

### Format on Save (clang-format Google style)

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "clang-format -style=Google -i \"$FILE_PATH\"",
        "description": "Format C++ files with Google style"
      }
    ]
  }
}
```

### cpplint Check

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "cpplint \"$FILE_PATH\" 2>&1 || true",
        "description": "Run cpplint on edited C++ files"
      }
    ]
  }
}
```

### clang-tidy Static Analysis

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "command": "clang-tidy --checks='google-*,modernize-*,bugprone-*,readability-*' \"$FILE_PATH\" -- -std=c++20 2>&1 | head -50",
        "description": "Run clang-tidy with Google checks on edited files"
      }
    ]
  }
}
```

## Stop Hooks

### Final Build Verification

```json
{
  "hooks": {
    "Stop": [
      {
        "command": "cmake --build build --config Release 2>&1 | tail -20",
        "description": "Verify production build at session end"
      }
    ]
  }
}
```

## Recommended CI Pipeline

Run these checks in order for every pull request:

```bash
# 1. Format check (fail if not formatted)
clang-format -style=Google --dry-run --Werror src/*.cpp src/*.h

# 2. cpplint
cpplint --recursive src/

# 3. clang-tidy static analysis
clang-tidy --checks='google-*,modernize-*,bugprone-*' src/*.cpp -- -std=c++20

# 4. cppcheck additional analysis
cppcheck --enable=all --suppress=missingInclude src/

# 5. Build
cmake -DCMAKE_BUILD_TYPE=Release -Bbuild . && cmake --build build

# 6. Tests with sanitizers
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" -Bbuild-asan .
cmake --build build-asan
ctest --test-dir build-asan --output-on-failure
```

## .clang-format Configuration

Create `.clang-format` at the project root:

```yaml
BasedOnStyle: Google
```

## .clang-tidy Configuration

Create `.clang-tidy` at the project root:

```yaml
Checks: >
  google-*,
  modernize-*,
  bugprone-*,
  readability-*,
  -modernize-use-trailing-return-type
WarningsAsErrors: "google-*"
```

## CPPLINT.cfg Configuration

Create `CPPLINT.cfg` at the project root:

```
set noparent
filter=-build/include_order
linelength=80
```
