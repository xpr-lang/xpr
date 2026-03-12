# XPR — Cross-Language Expression Language

A sandboxed expression language with JS/Python-familiar syntax, designed for data pipeline transforms.

## Repositories

| Repo | Description |
|------|-------------|
| [xpr-lang/xpr](https://github.com/xpr-lang/xpr) | Language specification and conformance tests (this repo) |
| [xpr-lang/xpr-js](https://github.com/xpr-lang/xpr-js) | JavaScript/TypeScript runtime |
| [xpr-lang/xpr-python](https://github.com/xpr-lang/xpr-python) | Python runtime |
| [xpr-lang/xpr-go](https://github.com/xpr-lang/xpr-go) | Go runtime |
| [xpr-lang/xpr-docs](https://github.com/xpr-lang/xpr-docs) | Documentation site |
| [xpr-lang/xpr-playground](https://github.com/xpr-lang/xpr-playground) | Interactive playground |

## Spec

See [XPR-SPEC.md](./XPR-SPEC.md) for the complete language specification.

## Conformance Tests

The `conformance/` directory contains YAML test files organized by category:

- `literals.yaml` — Number, string, boolean, null, array, object, template literals
- `arithmetic.yaml` — Arithmetic operators, precedence, associativity
- `comparison.yaml` — Equality and comparison operators
- `logic.yaml` — Logical operators, ternary, nullish coalescing
- `access.yaml` — Property access, optional chaining
- `functions.yaml` — Built-in functions, custom functions, arrow functions
- `strings.yaml` — String methods
- `collections.yaml` — Array collection methods
- `pipe.yaml` — Pipe operator

See [conformance/SCHEMA.md](./conformance/SCHEMA.md) for the test format specification.

## Using Conformance Tests in Your Runtime

Add this repo as a git submodule:

```bash
git submodule add https://github.com/xpr-lang/xpr.git conformance
```

Then read all `*.yaml` files from the `conformance/` directory in your test runner.

## License

MIT
