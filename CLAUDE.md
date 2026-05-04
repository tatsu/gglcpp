# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

**gglcpp** is a Claude Code plugin that distributes the Google C++ Style Guide as:
- A **skill** (`skills/google-cpp-style-guide/SKILL.md`) invokable via `/gglcpp:google-cpp-style-guide`
- **Rules** (`rules/cpp/*.md`) auto-loaded by Claude Code when C++ files are detected

There is no application code to build or test. All content is Markdown and configuration files.

## Repository Structure

```
skills/google-cpp-style-guide/SKILL.md   # Full style guide reference (invoke as a skill)
rules/cpp/
  coding-style.md   # Naming, headers, scoping, classes, functions — summary rules
  hooks.md          # Hook configs for clang-format, cpplint, clang-tidy
  patterns.md       # RAII, smart pointers, error handling, composition patterns
  security.md       # Memory safety, buffer safety, UB prevention, input validation
  testing.md        # GoogleTest setup, fixtures, mocking, sanitizers, coverage
.clang-format       # BasedOnStyle: Google (example config for downstream projects)
```

## Rules Frontmatter

Each file in `rules/cpp/` must include a `paths:` frontmatter block that declares which file patterns trigger auto-loading. Example:

```yaml
---
paths:
  - "**/*.cpp"
  - "**/*.cc"
  - "**/*.h"
---
```

Without this block, Claude Code will not auto-load the rule.

## Skill Frontmatter

`SKILL.md` requires these frontmatter fields:

```yaml
---
name: google-cpp-style-guide
description: <one-line summary>
origin: gglcpp
---
```

The `origin` field must match the plugin name so the skill appears as `/gglcpp:google-cpp-style-guide`.

## Rules vs. Skill Consistency

Rule files (`rules/cpp/*.md`) are condensed summaries; `SKILL.md` is the comprehensive reference. When updating content, check both: a correction to naming conventions in `coding-style.md` likely needs a matching fix in the Naming section of `SKILL.md`, and vice versa.

## Content Rules

- All content must be faithful to the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- Target: **C++20** only — no C++23, no non-standard extensions
- Code examples in the skill and rules must be correct C++20
- Rule files are summaries; the skill is the comprehensive reference
- License: CC-BY 3.0 (same as the source material)

## Install Path Convention

Rules are distributed under `gglcpp/cpp/` to avoid colliding with any existing `cpp/` rules the user may have:

```bash
# Global install
cp -r rules/cpp ~/.claude/rules/gglcpp/cpp

# Project-local install
cp -r rules/cpp .claude/rules/gglcpp/cpp
```

Claude Code discovers rules recursively, so `~/.claude/rules/gglcpp/cpp/*.md` is loaded the same way as `~/.claude/rules/cpp/*.md`.

## Checking Changes

Since there's no build system, verify changes by:

```bash
# Verify paths: frontmatter is present in all rule files
grep -L "^paths:" rules/cpp/*.md

# Verify skill frontmatter fields (name: and origin: must both be present)
grep -L "^name:" skills/google-cpp-style-guide/SKILL.md
grep -L "^origin:" skills/google-cpp-style-guide/SKILL.md
```
