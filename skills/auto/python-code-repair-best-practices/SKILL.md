---
name: python-code-repair-best-practices
description: Use when performing Python code repair tasks to ensure code correctness, testing, and documentation.
---
- Do not modify existing test files; add new test files for regression tests.
- Add type annotations to all public functions’ parameters and return values.
- Add a regression test function per bug fixed in a new tests/test_regressions.py file.
- Update CHANGELOG.md under '## Unreleased' with a bullet for each fix.
- Follow docstring specifications precisely and ensure code matches them.
