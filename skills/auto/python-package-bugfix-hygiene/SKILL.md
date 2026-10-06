---
name: python-package-bugfix-hygiene
description: Use when fixing bugs in a Python package.
---
1. Add type annotations to every parameter and return value of every public function (a name not starting with `_`).
2. Create `tests/test_regressions.py` with one test function for each bug fixed; include at least 3 test functions.
3. Record every fix in `CHANGELOG.md` under `## Unreleased`, using one bullet per fix in this exact format: `- fix(<function name>): <short description>`.
4. Run the tests and verify the regression test file passes.
5. Self-check:
   - Are all public functions fully annotated?
   - Does `tests/test_regressions.py` contain at least 3 tests, one per fix?
   - Does `CHANGELOG.md` contain at least 3 correctly formatted bullets under `## Unreleased`?
