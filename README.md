# XPR — Cross-Language Expression Language

A sandboxed expression language with JS/Python-familiar syntax, designed for data pipeline transforms.

## Repositories

| Repo | Description |
|------|-------------|
| [xpr-lang/xpr](https://github.com/xpr-lang/xpr) | Language specification and conformance tests (this repo) |
| [xpr-lang/xpr-js](https://github.com/xpr-lang/xpr-js) | JavaScript/TypeScript runtime |
| [xpr-lang/xpr-python](https://github.com/xpr-lang/xpr-python) | Python runtime |
| [xpr-lang/xpr-go](https://github.com/xpr-lang/xpr-go) | Go runtime |
| [xpr-lang/xpr-docs](https://xpr-lang.github.io/xpr-docs/) | Documentation site |
| [xpr-lang/xpr-playground](https://github.com/xpr-lang/xpr-playground) | Interactive playground |

## Spec

See [XPR-SPEC.md](./XPR-SPEC.md) for the complete language specification.

## Conformance Tests

The `conformance/` directory contains YAML test files organized by category:

- `access.yaml` — Property access, optional chaining
- `arithmetic.yaml` — Arithmetic operators, precedence, associativity
- `collections.yaml` — Array collection methods (v0.1)
- `collections_v5.yaml` — New array methods (v0.5): sortBy, take, drop, sum, avg, etc.
- `comparison.yaml` — Equality and comparison operators
- `datetime.yaml` — Date/time functions (v0.3)
- `destructuring.yaml` — Object and array destructuring (v0.4)
- `examples.yaml` — End-to-end integration examples
- `features_v5.yaml` — v0.5 features: math, type predicates, fromEntries, rest params
- `functions.yaml` — Built-in functions, custom functions, arrow functions
- `let.yaml` — Let bindings, scoping, shadowing, closures
- `literals.yaml` — Number, string, boolean, null, array, object, template literals
- `logic.yaml` — Logical operators, ternary, nullish coalescing
- `math_v5.yaml` — Math functions and constants (v0.5)
- `methods_v2.yaml` — v0.2 array, string, object methods and range()
- `negative_indexing.yaml` — Negative array indexing (v0.3)
- `pipe.yaml` — Pipe operator
- `regex.yaml` — Regex functions (v0.3): matches, match, matchAll, replacePattern
- `regex_literals.yaml` — Regex literal syntax and regex type (v0.4)
- `spread.yaml` — Array and object spread operator
- `strings.yaml` — String methods
- `type_predicates.yaml` — Type predicate functions (v0.5)

See [conformance/SCHEMA.md](./conformance/SCHEMA.md) for the test format specification.

## Using Conformance Tests in Your Runtime

Add this repo as a git submodule:

```bash
git submodule add https://github.com/xpr-lang/xpr.git conformance
```

Then read all `*.yaml` files from the `conformance/` directory in your test runner.

## License

MIT
