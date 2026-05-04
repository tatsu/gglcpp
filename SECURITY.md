# Security Policy

## Scope

This repository contains documentation and configuration files only (Markdown, YAML, JSON).
There is no executable code, so the attack surface is limited to:

- Malicious content embedded in Markdown that could be rendered unsafely by a downstream tool
- Hook command strings in `rules/cpp/hooks.md` that could be misused if copied verbatim

## Reporting a Vulnerability

If you discover a security concern in this repository (for example, a hook command that could cause harm when executed, or content that misrepresents safe C++ practices in a dangerous way), please report it privately before opening a public issue.

**Contact:** Open a [GitHub Security Advisory](https://github.com/tatsu/gglcpp/security/advisories/new) on this repository.

Please include:
- A description of the concern
- Which file and section is affected
- A suggested fix if you have one

We will respond within 7 days.

## Out of Scope

- Vulnerabilities in Claude Code itself — report those to Anthropic
- Vulnerabilities in cpplint, clang-format, or clang-tidy — report those to their respective projects
- General C++ security questions unrelated to the content of this repository
