# XPR Conformance Test Schema

## File Format

Each conformance test file is a YAML document with the following structure:

```yaml
suite: string       # Category name (e.g., "literals", "arithmetic")
version: "0.1.0"    # Spec version these tests conform to
tests:
  - name: string            # Unique test name within suite (required)
    expression: string      # XPR expression to evaluate (required)
    context: object?        # Optional evaluation context (default: {})
    expected: any?          # Expected result (mutually exclusive with error)
    error: string?          # Expected error message substring (mutually exclusive with expected)
    skip: boolean?          # Skip this test (default: false)
    tags: string[]?         # Optional tags for filtering (e.g., ["level-1", "core"])
```

## Rules

- Each test MUST have either `expected` OR `error`, but NOT both and NOT neither.
- `name` must be unique within a suite file.
- `context` defaults to `{}` if omitted.
- `error` is a substring match — the actual error message must CONTAIN the `error` string.
- `skip: true` causes the test to be skipped (not failed) by all runtimes.
- `tags` are informational only — runtimes may use them for filtering.

## Conformance Levels (via tags)

- `level-1` / `core`: Literals, arithmetic, comparison, logic, ternary, property access, function calls
- `level-2` / `collections`: map, filter, reduce, find, some, every, sort, arrow functions
- `level-3` / `advanced`: Pipe, template literals, optional chaining, nullish coalescing, object/array methods

## Runtime Consumption

Runtimes consume this test suite via git submodule:

```bash
git submodule add https://github.com/xpr-lang/xpr.git conformance
```

The conformance test runner reads all `*.yaml` files from the `conformance/` directory.
