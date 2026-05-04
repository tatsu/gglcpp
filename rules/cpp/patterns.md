---
paths:
  - "**/*.cpp"
  - "**/*.hpp"
  - "**/*.cc"
  - "**/*.hh"
  - "**/*.cxx"
  - "**/*.h"
---
# C++ Patterns (Google Style)

Design and implementation patterns aligned with the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html).

## RAII (Resource Acquisition Is Initialization)

Tie resource lifetime to object lifetime. Preferred over manual cleanup:

```cpp
class FileHandle {
 public:
  explicit FileHandle(const std::string& path)
      : file_(std::fopen(path.c_str(), "r")) {
    if (!file_) throw std::runtime_error("Cannot open: " + path);
  }
  ~FileHandle() { if (file_) std::fclose(file_); }
  FileHandle(const FileHandle&) = delete;
  FileHandle& operator=(const FileHandle&) = delete;

 private:
  std::FILE* file_;
};
```

## Rule of Zero / Rule of Five

**Rule of Zero**: prefer classes that need no custom destructor, copy/move constructors, or assignments. Let the compiler generate them from member types.

**Rule of Five**: if you define *any* of {destructor, copy ctor, copy assign, move ctor, move assign}, define or explicitly delete all five.

```cpp
// Rule of Zero: all generated from string members
struct Config {
  std::string host;
  int port = 0;
};

// Rule of Five: manages a raw resource
class Buffer {
 public:
  explicit Buffer(std::size_t size);
  ~Buffer();
  Buffer(const Buffer&);
  Buffer& operator=(const Buffer&);
  Buffer(Buffer&&) noexcept;
  Buffer& operator=(Buffer&&) noexcept;
};
```

## Smart Pointer Ownership

```cpp
// Exclusive ownership
auto widget = std::make_unique<Widget>("config");

// Shared ownership (only when truly needed)
auto cache = std::make_shared<Cache>(1024);

// Non-owning observer: raw pointer or reference
void Render(const Widget* w);  // does NOT own w
void Display(const Widget& w); // does NOT own w
```

Pass `std::unique_ptr` by value to transfer ownership. Return `std::unique_ptr` from factory functions.

## Value Semantics

- Pass small/trivial types by value
- Pass large types by `const&`
- Return by value (rely on NRVO)
- Use move semantics for sink parameters

```cpp
// GOOD: return by value, let NRVO optimize
std::vector<int> BuildList(int n) {
  std::vector<int> result;
  result.reserve(n);
  // ...
  return result;
}
```

## Error Handling Without Exceptions

Google style prefers not using exceptions. Use these patterns instead:

```cpp
// std::optional for values that may be absent
std::optional<Config> ParseConfig(std::string_view input);

// absl::Status / absl::StatusOr for operations that can fail
absl::StatusOr<ParseResult> Parse(std::string_view input);

// Returning error codes for performance-critical code
enum class Status { kOk, kNotFound, kInvalidArg };
Status DoWork(Input in, Output* out);
```

## Interfaces via Abstract Base Classes

```cpp
// Pure interface
class StorageBackend {
 public:
  virtual ~StorageBackend() = default;
  virtual absl::Status Write(std::string_view key,
                             std::string_view value) = 0;
  virtual absl::StatusOr<std::string> Read(std::string_view key) = 0;
};

// Concrete implementation
class FileStorage final : public StorageBackend {
 public:
  absl::Status Write(std::string_view key,
                     std::string_view value) override;
  absl::StatusOr<std::string> Read(std::string_view key) override;
};
```

## Prefer Composition Over Inheritance

```cpp
// BAD: deep inheritance for code reuse
class LoggingFileStorage : public FileStorage { /* ... */ };

// GOOD: compose the logger
class LoggingStorage : public StorageBackend {
 public:
  explicit LoggingStorage(std::unique_ptr<StorageBackend> inner,
                          Logger* logger);
  absl::Status Write(std::string_view key,
                     std::string_view value) override;
 private:
  std::unique_ptr<StorageBackend> inner_;
  Logger* logger_;  // non-owning
};
```

## Concepts for Template Constraints (C++20)

```cpp
#include <concepts>

// Constrain with standard concepts
template<std::integral T>
T Clamp(T value, T lo, T hi);

// Custom concept
template<typename T>
concept Printable = requires(const T& t) {
  { t.ToString() } -> std::convertible_to<std::string>;
};

template<Printable T>
void Log(const T& obj);
```

## Thread Safety

Always use RAII locks:

```cpp
class ThreadSafeCounter {
 public:
  void Increment() {
    std::lock_guard<std::mutex> lock(mu_);
    ++count_;
  }
  int Get() const {
    std::lock_guard<std::mutex> lock(mu_);
    return count_;
  }
 private:
  mutable std::mutex mu_;
  int count_ = 0;
};
```

## Prefer `absl` Utilities

Google codebases use Abseil for portable, well-tested utilities:

- `absl::string_view` -> `std::string_view` (C++17)
- `absl::flat_hash_map` instead of `std::unordered_map` (better performance)
- `absl::Status` / `absl::StatusOr` for error handling
- `absl::StrCat`, `absl::StrFormat` for string formatting

## Reference

See skill `gglcpp:google-cpp-style-guide` for full naming, formatting, and feature-usage rules.
