---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
  - "**/CMakeLists.txt"
---
# C++ Coding Style (Google)

C++ coding standards following the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html). Target: **C++20**.

## C++ Version

Target C++20. Do not use C++23 features or non-standard extensions. Consider portability before using C++17/20 features in a new project.

## Header Files

- Every `.cc` file has an associated `.h` file
- Headers are self-contained (compile on their own)
- `#define` guard format: `<PROJECT>_<PATH>_<FILE>_H_`
- Include only what you use — no transitive inclusion reliance
- Avoid forward declarations of entities your project does not own
- Inline function bodies in headers: 10 lines or fewer

```cpp
#ifndef MYPROJECT_FOO_BAR_H_
#define MYPROJECT_FOO_BAR_H_

#include <string>

namespace myproject {

class Bar {
 public:
  explicit Bar(std::string name);
  const std::string& name() const;

 private:
  std::string name_;
};

}  // namespace myproject

#endif  // MYPROJECT_FOO_BAR_H_
```

## Include Order

1. Related header (`foo.cc` -> `foo.h` first)
2. C system headers (`<unistd.h>`)
3. C++ standard library headers (`<string>`, `<vector>`)
4. Third-party library headers
5. Your project headers

Groups separated by blank lines; alphabetical within each group.

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Types (class, struct, enum, alias) | `PascalCase` | `UrlTable` |
| Functions | `PascalCase` | `AddTableEntry()` |
| Variables (local, parameters) | `snake_case` | `table_name` |
| Class members | `snake_case_` (trailing `_`) | `num_entries_` |
| Constants (`constexpr`, `const` with static storage) | `kPascalCase` | `kMaxConnections` |
| Namespaces | `lowercase` | `mynamespace` |
| Enumerators | `kPascalCase` | `kOptionA` |
| Macros | `ALL_CAPS` | `MY_MACRO` |

## Scoping

- Use namespaces; end with `// namespace foo`
- No `using namespace` at global scope in headers
- Prefer unnamed namespaces or `static` for internal linkage
- Declare variables as close to first use as possible; initialize at declaration

## Classes

- Mark single-argument constructors `explicit`
- Rule of Zero or Rule of Five — never define some but not all
- Base class destructors: `public virtual` or `protected non-virtual`
- Use `override` on overrides; never use `virtual` on an override
- Prefer `private` data; expose via accessors
- Declaration order: `public`, `protected`, `private`

## Functions

- Keep functions short (<= 40 lines)
- Return values instead of output parameters
- Pass cheap types by value, expensive types by `const&`
- Default arguments only on non-virtual functions, trailing positions only

## Modern C++ Usage

- `nullptr` for null pointers, never `0` or `NULL`
- `constexpr` for compile-time constants (not macros)
- `static_cast<T>` for conversions, never C-style `(T)x`
- `auto` when type is obvious from context; explicit type otherwise
- `enum class` over plain `enum`

## Resource Management

- No raw `new`/`delete` — use `std::unique_ptr` / `std::make_unique`
- `std::shared_ptr` only when ownership is genuinely shared
- Raw pointer = non-owning observer only

## What to Avoid

- Exceptions (unless the project already uses them)
- RTTI (`dynamic_cast`, `typeid`)
- C-style casts
- `using namespace` at global scope in headers
- C++20 modules (`module`, `export`, `import`)
- Custom coroutine promise types
- `std::bind` — use lambdas instead
- `std::regex` — use RE2 or project-approved alternatives

## Formatting Summary

- Line length: **80 characters**
- Indent: **2 spaces**, no tabs
- Opening braces on the same line
- Always use braces for control flow bodies
- `*` and `&` attach to the type: `char* ptr`

## Reference

See skill `gglcpp:google-cpp-style-guide` for comprehensive rules and examples.
