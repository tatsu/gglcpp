# gglcpp — Google C++ Style Guide Plugin for Claude Code

A [Claude Code](https://claude.ai/code) plugin that brings the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html) into your AI-assisted C++ development workflow.

Includes:
- **Skill** — comprehensive reference: naming, formatting, headers, classes, functions, feature restrictions
- **Rules** — auto-loaded coding style, hooks, patterns, security, and testing guidance for C++ files

---

## Marketplace Install (Skill)

### Step 1 — Add the marketplace

```bash
claude plugin marketplace add https://github.com/tatsu/gglcpp
```

### Step 2 — Install the skill

```bash
claude plugin install google-cpp-style-guide@gglcpp
```

### Step 3 — Use the skill

In any Claude Code session, invoke it with:

```
/gglcpp:google-cpp-style-guide
```

Or reference it in your prompts:

```
Review this C++ code against the Google C++ Style Guide.
```

---

## Rules Install (Manual Copy)

> **Note**: Claude Code does not automatically install rules from plugins. You must copy the rule files manually into your project or global rules directory.

Rules are installed under `gglcpp/cpp/` to avoid conflicts with any existing `cpp/` rules you may have in place.

### Option A — curl (project-level)

```bash
mkdir -p .claude/rules/gglcpp/cpp
for f in coding-style hooks patterns security testing; do
  curl -fsSL \
    "https://raw.githubusercontent.com/tatsu/gglcpp/main/rules/cpp/${f}.md" \
    -o ".claude/rules/gglcpp/cpp/${f}.md"
done
```

### Option B — curl (global, all projects)

```bash
mkdir -p ~/.claude/rules/gglcpp/cpp
for f in coding-style hooks patterns security testing; do
  curl -fsSL \
    "https://raw.githubusercontent.com/tatsu/gglcpp/main/rules/cpp/${f}.md" \
    -o "${HOME}/.claude/rules/gglcpp/cpp/${f}.md"
done
```

### Option C — clone and copy

```bash
git clone https://github.com/tatsu/gglcpp.git
# Project-local:
cp -r gglcpp/rules/cpp .claude/rules/gglcpp/cpp
# Or global:
cp -r gglcpp/rules/cpp ~/.claude/rules/gglcpp/cpp
```

---

## What's Included

### Skill

| File | Description |
|------|-------------|
| `skills/google-cpp-style-guide/SKILL.md` | Full Google C++ Style Guide reference with examples |

Invoke as `/gglcpp:google-cpp-style-guide` after installing.

### Rules (auto-loaded by Claude Code for C++ files)

| File | Description |
|------|-------------|
| `rules/cpp/coding-style.md` | Naming, headers, classes, formatting summary |
| `rules/cpp/hooks.md` | cpplint, clang-format, clang-tidy hook configurations |
| `rules/cpp/patterns.md` | RAII, smart pointers, error handling, composition |
| `rules/cpp/security.md` | Memory safety, buffer safety, UB prevention, input validation |
| `rules/cpp/testing.md` | GoogleTest setup, fixtures, mocking, sanitizers, coverage |

Rules are loaded automatically when Claude Code detects C++ files (`*.cpp`, `*.h`, `*.cc`, etc.).

---

## Tooling Setup

### clang-format (Google style)

Create `.clang-format` at your project root:

```yaml
BasedOnStyle: Google
```

Apply:

```bash
clang-format -style=Google -i src/**/*.cpp src/**/*.h
```

### cpplint

```bash
pip install cpplint
cpplint --recursive src/
```

### clang-tidy

Create `.clang-tidy` at your project root:

```yaml
Checks: "google-*,modernize-*,bugprone-*,readability-*"
WarningsAsErrors: "google-*"
```

Run:

```bash
clang-tidy src/*.cpp -- -std=c++20
```

---

## C++ Version

This plugin targets **C++20**. C++23 features and non-standard extensions are not covered.

---

## Source Material

Rules and skill content is derived from the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html), which is licensed under [CC-BY 3.0](https://creativecommons.org/licenses/by/3.0/).

---

## License

[CC-BY 3.0](LICENSE) — same license as the Google C++ Style Guide.

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md).

## Security

See [SECURITY.md](SECURITY.md).
