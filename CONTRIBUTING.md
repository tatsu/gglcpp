# Contributing to gglcpp

Thank you for your interest in contributing!

## What to Contribute

- Corrections to rule or skill content that diverges from the [Google C++ Style Guide](https://google.github.io/styleguide/cppguide.html)
- Missing sections from the style guide not yet covered
- Improved code examples
- Hook configurations for additional C++ tooling

## Ground Rules

- All content must remain faithful to the Google C++ Style Guide
- Target C++20; do not add C++23 or non-standard content
- Keep rule files focused and under 800 lines
- Code examples must compile cleanly with a C++20-conformant compiler

## How to Submit Changes

1. Fork the repository
2. Create a branch: `git checkout -b fix/description`
3. Make your changes
4. Verify that examples in the changed files are correct
5. Submit a pull request with a clear description of what changed and why

## Commit Message Format

```
fix: correct lambda capture example in SKILL.md
feat: add ranges section to coding-style rule
docs: update clang-tidy check list in hooks.md
```

Types: `feat`, `fix`, `docs`, `chore`

## License

By contributing, you agree that your contributions will be licensed under [CC-BY 3.0](LICENSE), consistent with the source material.
