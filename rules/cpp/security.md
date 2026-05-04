---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
---
# C++ Security (Google Style)

Security rules for C++ codebases following the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html).

## Memory Safety

- Never use raw `new`/`delete` — use `std::make_unique` / `std::make_shared`
- Never use C-style arrays for owned buffers — use `std::vector` or `std::array`
- Never use `malloc`/`free` — use C++ allocation
- Avoid `reinterpret_cast` unless absolutely necessary and thoroughly documented

```cpp
// BAD: manual memory management
char* buf = new char[size];
delete[] buf;  // may leak or double-free

// GOOD: automatic lifetime
std::vector<char> buf(size);
```

## Buffer Safety

- Use `std::string` over `char*` for string data
- Use `.at()` for bounds-checked container access when the index may be invalid
- Never use `strcpy`, `strcat`, `sprintf` — use `std::string`, `absl::StrCat`, or `absl::StrFormat`
- Never use `gets` — removed from C++14 and always unsafe

```cpp
// BAD: unbounded copy
char dest[64];
strcpy(dest, src);  // buffer overflow if src > 63 chars

// GOOD: safe string handling
std::string dest = src;
```

## Undefined Behavior Prevention

- Always initialize variables at declaration
- Never dereference null or dangling pointers
- Avoid signed integer overflow — use `int64_t` when overflow is possible
- Never access an object after its lifetime ends (dangling reference/pointer)

```cpp
// BAD: uninitialized variable
int count;
if (condition) count = 5;
use(count);  // undefined if !condition

// GOOD: always initialize
int count = 0;
if (condition) count = 5;
use(count);
```

## Type Safety

- Never use C-style casts — use `static_cast`, `const_cast`, or `reinterpret_cast` with a comment
- Never cast away `const` unless you own the object and know it is truly mutable
- Avoid `void*` in new APIs — use templates or `std::any` instead

```cpp
// BAD: C-style cast hides errors
int* p = (int*)some_void_ptr;

// GOOD: explicit and auditable
int* p = static_cast<int*>(some_void_ptr);
```

## Input Validation

Validate all inputs at system boundaries (user input, network data, file content, IPC):

```cpp
absl::StatusOr<Config> ParseConfig(std::string_view raw) {
  if (raw.empty()) return absl::InvalidArgumentError("empty config");
  if (raw.size() > kMaxConfigSize) {
    return absl::InvalidArgumentError("config too large");
  }
  // parse...
}
```

## No Secrets in Source

- Never hardcode passwords, API keys, tokens, or credentials
- Use environment variables or a secrets manager at runtime
- Verify required secrets exist at startup and fail fast with a clear error

## Integer Overflow

```cpp
// BAD: silent overflow
int total = a + b;  // wraps silently if > INT_MAX

// GOOD: explicit check before operation
if (a > std::numeric_limits<int>::max() - b) {
  return absl::OutOfRangeError("integer overflow");
}
int total = a + b;
```

## Static Analysis in CI

```bash
# Sanitizers: address, undefined behavior
cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined" -Bbuild-asan .
cmake --build build-asan && ctest --test-dir build-asan

# clang-tidy with security-relevant checks
clang-tidy --checks='bugprone-*,cert-*,clang-analyzer-security.*,google-*' \
  src/*.cpp -- -std=c++20

# cppcheck
cppcheck --enable=all --error-exitcode=1 src/
```

## Mandatory Checks Before Commit

- [ ] No raw `new`/`delete` — smart pointers used
- [ ] No `strcpy`, `strcat`, `sprintf`, `gets`
- [ ] No hardcoded secrets (API keys, passwords, tokens)
- [ ] All user/external inputs validated before use
- [ ] No C-style casts
- [ ] All variables initialized at declaration
- [ ] No signed integer operations that could overflow without a check
- [ ] `clang-tidy` and `cppcheck` pass without new warnings

## Reference

See skill `gglcpp:google-cpp-style-guide` for the full guide including casting rules, integer type guidance, and macro avoidance.
