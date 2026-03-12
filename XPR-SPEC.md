# XPR — Cross-Language Expression Language

A sandboxed expression language with JS/Python-familiar syntax, designed for data pipeline transforms, with native interpreters in JavaScript, Python, and Go.

**Version**: 0.1.0  
**Status**: Draft

---

## Problem

Existing expression languages fail for cross-language data pipeline use cases:

| Existing | Problem |
|---|---|
| JEXL / PyJEXL | No lambdas, no collection ops, dead-simple grammar |
| CEL (Google) | Policy-oriented, not data-transform. Verbose. No JS runtime. |
| JSONata | JSON-specific, weird syntax, no Python/Go |
| Starlark | Python-like only, no JS feel, designed for configs |
| Expr (Go) | Go-only |

Nobody owns **"modern-feeling expression language for data pipelines, cross-runtime."**

---

## Non-Goals

- Not a general-purpose language. No `while`, no `class`, no I/O, no imports.
- Not a transpiler. Doesn't compile to JS/Python/Go — it **evaluates** identically across all three.
- No side effects. Pure expressions only.
- No variable assignment (`let`, `const`, `var`).
- No loops (`for`, `while`). Collection operations are the only iteration.

---

## Type System

XPR has **6 primitive types**:

| Type | Description | Examples |
|------|-------------|---------|
| `number` | IEEE 754 double-precision float | `42`, `3.14`, `-1`, `1e10` |
| `string` | UTF-8 text | `"hello"`, `'world'` |
| `boolean` | True or false | `true`, `false` |
| `null` | Absence of value | `null` |
| `array` | Ordered list | `[1, 2, 3]`, `[]` |
| `object` | Key-value map | `{"a": 1}`, `{}` |

### No `undefined`

XPR uses `null` exclusively. There is no `undefined` type.

### No `date` in v0.1

Date/time operations are deferred to v0.2 due to timezone and format complexity. See [Future](#future-v02).

### Number Representation

`number` is IEEE 754 double-precision floating point (same as JavaScript). There is no separate integer type. No BigInt. No hex/octal/binary literals.

### Truthiness

The following values are **falsy**:
- `false`
- `null`
- `0`
- `""` (empty string)

Everything else is **truthy**, including:
- `[]` (empty array)
- `{}` (empty object)
- `"0"` (non-empty string)
- Any non-zero number

### Type Coercion

XPR uses **strict typing** — no implicit coercion between types.

| Expression | Result |
|-----------|--------|
| `"5" + 1` | **Error**: Type error — cannot add string and number |
| `true * 2` | **Error**: Type error — cannot multiply boolean and number |
| `"hello" + " world"` | `"hello world"` — string concatenation is valid |
| `int("5") + 1` | `6` — explicit conversion is valid |

### Null Semantics

| Expression | Result |
|-----------|--------|
| `null + 1` | **Error**: Type error |
| `null == null` | `true` |
| `null == 0` | `false` (strict equality, no coercion) |
| `null == false` | `false` |
| `null ?? "default"` | `"default"` |
| `0 ?? "default"` | `0` (0 is not null) |
| `"" ?? "default"` | `""` (empty string is not null) |

### Missing Property Access

Accessing a property that doesn't exist returns `null` (not an error):

```
user.email        // null if email doesn't exist on user
user.address.zip  // null if address doesn't exist
```

However, accessing a property on `null` (without optional chaining) IS an error:

```
user.foo.bar.baz  // error if user.foo is null — cannot access .bar on null
user?.foo?.bar    // null — optional chaining propagates null safely
```

### Array Out of Bounds

Accessing an array index that doesn't exist returns `null`:

```
items[99]   // null if items has fewer than 100 elements
items[-1]   // error — negative indexing not supported in v0.1
```

### Division by Zero

Division by zero is an error (not `Infinity`):

```
1 / 0   // Error: Division by zero
```

---

## Syntax

### Literals

```javascript
// Numbers
42, 3.14, -1, 0, 1e10, 2.5e-3

// Strings (single or double quotes)
"hello", 'world', "", ''

// Escape sequences in strings
"\n"  // newline
"\t"  // tab
"\\"  // backslash
"\'"  // single quote
"\""  // double quote
"\r"  // carriage return
"\0"  // null character

// Booleans
true, false

// Null
null

// Arrays
[1, 2, 3], [], [1, "mixed", true, null]

// Objects
{"key": "value"}, {}, {"nested": {"deep": true}}

// Template literals (backtick)
`Hello ${name}, you have ${count} items`
`Total: ${price * qty}`
`plain string`
```

### Operators

```javascript
// Arithmetic
+, -, *, /, %, **

// Comparison
==, !=, <, >, <=, >=

// Logical
&&, ||, !

// Special
?? (nullish coalescing)
?. (optional chaining)
|> (pipe)
```

### Ternary

```javascript
age >= 18 ? "adult" : "minor"
```

### Arrow Functions

```javascript
// Single parameter (no parens needed)
x => x * 2

// Multiple parameters (parens required)
(x, y) => x + y

// Used as arguments to collection methods
items.map(x => x.price * x.qty)
items.filter(x => x.active && x.price > 10)
items.sort((a, b) => a.price - b.price)
```

### Collection Operations

```javascript
items.map(x => x.price * x.qty)
items.filter(x => x.active && x.price > 10)
items.reduce((sum, x) => sum + x, 0)
items.find(x => x.id == targetId)
items.some(x => x.price > 100)
items.every(x => x.active)
items.flatMap(x => x.tags)
items.sort((a, b) => a.price - b.price)
items.reverse()
items.length
```

### String Operations

```javascript
name.upper()
name.lower()
name.trim()
name.startsWith("Dr.")
name.endsWith("Jr.")
name.contains("admin")
name.split(",")
name.replace("old", "new")
name.slice(0, 5)
name.len()
```

### Math Functions

```javascript
round(3.7)    // 4
floor(3.7)    // 3
ceil(3.2)     // 4
abs(-5)       // 5
min(a, b)
max(a, b)
```

### Type Functions

```javascript
type(value)     // "number", "string", "boolean", "null", "array", "object"
int("42")       // 42
float("3.14")   // 3.14
string(42)      // "42"
bool(1)         // true
bool(0)         // false
```

### Object/Array Access

```javascript
user.address.city
users[0].name
{"a": 1, "b": 2}.keys()    // ["a", "b"]
{"a": 1, "b": 2}.values()  // [1, 2]
```

### Optional Chaining

```javascript
user?.address?.city    // null if user or address is null
obj.fn?.(arg)          // null if fn is null, otherwise calls fn(arg)
```

### Pipe Operator

```javascript
data |> filter(x => x.active) |> map(x => x.name) |> sort()
```

---

## Operator Precedence

Operators listed from **lowest** to **highest** precedence:

| Level | Operators | Associativity | Description |
|-------|-----------|---------------|-------------|
| 1 | `\|>` | left | Pipe |
| 2 | `? :` | right | Ternary conditional |
| 3 | `??` | left | Nullish coalescing |
| 4 | `\|\|` | left | Logical OR |
| 5 | `&&` | left | Logical AND |
| 6 | `== !=` | left | Equality |
| 7 | `< > <= >=` | left | Comparison |
| 8 | `+ -` | left | Addition/subtraction |
| 9 | `* / %` | left | Multiplication/division/modulo |
| 10 | `**` | right | Exponentiation |
| 11 | `! -` (unary) | prefix | Unary operators |
| 12 | `() [] . ?.` | left | Call, index, member, optional member |

### Precedence Examples

```javascript
2 + 3 * 4        // 14 (not 20) — * binds tighter than +
2 ** 3 ** 2      // 512 (not 64) — ** is right-associative: 2 ** (3 ** 2) = 2 ** 9
(2 + 3) * 4      // 20 — parens override precedence
1 + 2 |> string  // "3" — pipe is lowest, so (1 + 2) |> string
a |> f ? b : c   // (a |> f) ? b : c — pipe lower than ternary
```

---

## Pipe Operator

XPR uses **F#-style pipe** (not Hack-style):

### Semantics

- `x |> f` desugars to `f(x)` — LHS becomes the first argument of the RHS call
- `x |> f(a)` desugars to `f(x, a)` — LHS is prepended to the argument list
- `x |> f |> g` desugars to `g(f(x))` — left-to-right evaluation
- Pipe has the **lowest precedence** (below ternary)

### Examples

```javascript
// Basic pipe
3.7 |> round                    // round(3.7) → 4

// Pipe with additional args
"hello world" |> split(" ")     // split("hello world", " ") → ["hello", "world"]

// Chained pipe
[3,1,2] |> sort((a,b) => a - b) |> reverse()   // [3, 2, 1]

// Complex pipeline
items
  |> filter(x => x.active)
  |> map(x => x.name)
  |> sort()
```

### NOT Hack-style

XPR does NOT use a topic reference (`%`, `^`, `#`). The LHS is always the first argument.

---

## Arrow Functions

Arrow functions are the mechanism for passing behavior to collection methods.

### Syntax

```javascript
// Single parameter — no parens needed
x => x * 2

// Multiple parameters — parens required
(x, y) => x + y
(sum, item) => sum + item.price
```

### Rules

- Body is a **single expression** — no multi-line bodies, no statements
- Arrow functions are **not first-class values** in v0.1 — they only appear as arguments to collection methods and pipe targets
- Arrow binds tighter than pipe: `items |> filter(x => x > 5)` — the `=>` captures `x > 5`, not the whole pipe
- Parameters are scoped to the arrow function body — they shadow outer context variables

---

## Built-in Functions

### String Methods

Called on string values using dot notation:

| Method | Signature | Description |
|--------|-----------|-------------|
| `len` | `() → number` | Length of string in characters |
| `upper` | `() → string` | Uppercase |
| `lower` | `() → string` | Lowercase |
| `trim` | `() → string` | Remove leading/trailing whitespace |
| `startsWith` | `(s: string) → boolean` | True if string starts with s |
| `endsWith` | `(s: string) → boolean` | True if string ends with s |
| `contains` | `(s: string) → boolean` | True if string contains s |
| `split` | `(sep: string) → array` | Split by separator |
| `replace` | `(old: string, new: string) → string` | Replace ALL occurrences of old with new |
| `slice` | `(start: number, end?: number) → string` | Substring from start to end (exclusive) |

Calling a string method on a non-string value is an error.

### Math Functions

Global functions (not called on a value):

| Function | Signature | Description |
|----------|-----------|-------------|
| `round` | `(n: number) → number` | Round to nearest integer |
| `floor` | `(n: number) → number` | Round down |
| `ceil` | `(n: number) → number` | Round up |
| `abs` | `(n: number) → number` | Absolute value |
| `min` | `(a: number, b: number) → number` | Minimum of two numbers |
| `max` | `(a: number, b: number) → number` | Maximum of two numbers |

### Type Functions

Global functions for type inspection and conversion:

| Function | Signature | Description |
|----------|-----------|-------------|
| `type` | `(v: any) → string` | Returns type name: `"number"`, `"string"`, `"boolean"`, `"null"`, `"array"`, `"object"` |
| `int` | `(v: any) → number` | Convert to integer (truncates decimals) |
| `float` | `(v: any) → number` | Convert to float |
| `string` | `(v: any) → string` | Convert to string |
| `bool` | `(v: any) → boolean` | Convert to boolean (uses truthiness rules) |

### Collection Methods

Called on array values using dot notation:

| Method | Signature | Description |
|--------|-----------|-------------|
| `map` | `(fn: arrow) → array` | Transform each element |
| `filter` | `(fn: arrow) → array` | Keep elements where fn returns truthy |
| `reduce` | `(fn: arrow, init: any) → any` | Accumulate to single value |
| `find` | `(fn: arrow) → any \| null` | First element where fn returns truthy, or null |
| `some` | `(fn: arrow) → boolean` | True if any element satisfies fn |
| `every` | `(fn: arrow) → boolean` | True if all elements satisfy fn |
| `flatMap` | `(fn: arrow) → array` | Map then flatten one level |
| `sort` | `(fn?: arrow) → array` | Sort (optional comparator arrow) |
| `reverse` | `() → array` | Reverse order |

**Collection property** (accessed as property, not method):

| Property | Type | Description |
|----------|------|-------------|
| `.length` | `number` | Number of elements in array |

Calling a collection method on a non-array value is an error.

### Object Methods

Called on object values using dot notation:

| Method | Signature | Description |
|--------|-----------|-------------|
| `keys` | `() → array` | Array of property names |
| `values` | `() → array` | Array of property values |

---

## Error Model

### Fail-Fast

XPR uses fail-fast error handling. The first error encountered stops evaluation immediately. There is no error recovery, no partial results.

### Error Type

```
XprError {
  message: string    // Human-readable error description
  position: number   // Character offset from start of expression (0-indexed)
}
```

### Error Categories

**Parse errors** — detected during parsing:
- `"Unexpected token '${token}' at position ${pos}"`
- `"Unexpected end of expression at position ${pos}"`
- `"Unmatched '(' at position ${pos}"`

**Evaluation errors** — detected during evaluation:
- `"Division by zero at position ${pos}"`
- `"Type error: cannot add string and number at position ${pos}"`
- `"Type error: cannot call method '${name}' on ${type} at position ${pos}"`
- `"Unknown function '${name}' at position ${pos}"`
- `"Wrong number of arguments for '${name}': expected ${n}, got ${m} at position ${pos}"`

**Security errors** — detected during evaluation:
- `"Access denied: '${name}' is a restricted property at position ${pos}"`
- `"Expression depth limit exceeded at position ${pos}"`
- `"Expression timeout exceeded"`

---

## EBNF Grammar

Formal grammar for XPR v0.1. Uses standard EBNF notation:
- `*` = zero or more
- `+` = one or more
- `?` = optional
- `|` = alternation
- `( )` = grouping
- `[ ]` = character class

```ebnf
(* Top-level *)
Expression     ::= PipeExpr

(* Pipe — lowest precedence, left-associative *)
PipeExpr       ::= TernaryExpr ( "|>" TernaryExpr )*

(* Ternary — right-associative *)
TernaryExpr    ::= NullishExpr ( "?" Expression ":" TernaryExpr )?

(* Nullish coalescing *)
NullishExpr    ::= OrExpr ( "??" OrExpr )*

(* Logical OR *)
OrExpr         ::= AndExpr ( "||" AndExpr )*

(* Logical AND *)
AndExpr        ::= EqualityExpr ( "&&" EqualityExpr )*

(* Equality *)
EqualityExpr   ::= CompareExpr ( ( "==" | "!=" ) CompareExpr )*

(* Comparison *)
CompareExpr    ::= AddExpr ( ( "<" | ">" | "<=" | ">=" ) AddExpr )*

(* Addition and subtraction *)
AddExpr        ::= MulExpr ( ( "+" | "-" ) MulExpr )*

(* Multiplication, division, modulo *)
MulExpr        ::= ExpExpr ( ( "*" | "/" | "%" ) ExpExpr )*

(* Exponentiation — right-associative *)
ExpExpr        ::= UnaryExpr ( "**" ExpExpr )?

(* Unary operators *)
UnaryExpr      ::= ( "!" | "-" ) UnaryExpr
               | PostfixExpr

(* Postfix: member access, calls, indexing *)
PostfixExpr    ::= Primary PostfixOp*

PostfixOp      ::= "." Identifier                    (* member access *)
               | "?." Identifier                     (* optional member access *)
               | "[" Expression "]"                  (* computed index *)
               | "(" ArgList? ")"                    (* function call *)
               | "?." "(" ArgList? ")"               (* optional call *)

(* Primary expressions *)
Primary        ::= Literal
               | Identifier
               | "(" Expression ")"
               | ArrayLiteral
               | ObjectLiteral
               | TemplateLiteral
               | ArrowFunction

(* Arrow functions *)
ArrowFunction  ::= Identifier "=>" Expression
               | "(" ParamList? ")" "=>" Expression

ParamList      ::= Identifier ( "," Identifier )*

ArgList        ::= Expression ( "," Expression )*

(* Literals *)
ArrayLiteral   ::= "[" ( Expression ( "," Expression )* )? "]"

ObjectLiteral  ::= "{" ( Property ( "," Property )* )? "}"

Property       ::= ( Identifier | StringLiteral ) ":" Expression

TemplateLiteral ::= "`" TemplateContent* "`"

TemplateContent ::= TemplateChar+
                | "${" Expression "}"

(* Atomic literals *)
Literal        ::= NumberLiteral
               | StringLiteral
               | BooleanLiteral
               | NullLiteral

NumberLiteral  ::= [0-9]+ ( "." [0-9]+ )? ( ( "e" | "E" ) ( "+" | "-" )? [0-9]+ )?

StringLiteral  ::= '"' StringChar* '"'
               | "'" StringChar* "'"

StringChar     ::= [^"'\\\n]
               | "\\" EscapeSeq

EscapeSeq      ::= '"' | "'" | "\\" | "n" | "t" | "r" | "0"

BooleanLiteral ::= "true" | "false"

NullLiteral    ::= "null"

Identifier     ::= [a-zA-Z_] [a-zA-Z0-9_]*

TemplateChar   ::= [^`$\\]
               | "\\" .
               | "$" [^{]
```

---

## Security and Sandbox

### Blocked Property Names

The following property names are blocked and will throw `XprError` if accessed:

- `__proto__`
- `constructor`
- `prototype`

This prevents prototype pollution and access to host language internals.

### Limits

| Limit | Default | Configurable |
|-------|---------|-------------|
| Expression timeout | 100ms | Yes |
| Max AST depth | 50 | Yes |
| Max expression length | 10,000 characters | Yes |

### Context Immutability

The context object passed to `evaluate()` is **read-only**. Expressions cannot mutate input data.

### No Host Access

Expressions have no access to:
- Host filesystem
- Network
- Environment variables
- Host language globals (no `eval`, no `require`, no `import`)

---

## Context Model

Expressions evaluate against a **context object** — the data passed in by the host application.

```python
# Python host
engine = xpr.Xpr()
result = engine.evaluate(
    "items.filter(x => x.price > threshold).map(x => x.name)",
    context={"items": [...], "threshold": 50}
)
```

```javascript
// JavaScript host
const engine = new Xpr();
const result = engine.evaluate(
    'items.filter(x => x.price > threshold).map(x => x.name)',
    { items: [...], threshold: 50 }
);
```

```go
// Go host
engine := xpr.New()
result, err := engine.Evaluate(
    `items.filter(x => x.price > threshold).map(x => x.name)`,
    map[string]any{"items": []any{...}, "threshold": 50},
)
```

---

## Custom Functions

Host applications can register domain-specific functions:

```python
engine.add_function("slugify", lambda s: s.lower().replace(" ", "-"))
# Then usable in expressions:
# slugify(product.name)
```

```javascript
engine.addFunction("slugify", (s) => s.toLowerCase().replace(/ /g, "-"));
// Then usable in expressions:
// slugify(product.name)
```

Custom functions are called like built-in functions. They receive evaluated arguments and must return a value.

---

## AST Node Types

17 node types. ESTree-influenced, simplified for expression-only context.

Every node carries a `position` field (character offset) for error reporting.

```
── Literals (6) ──────────────────────────────
NumberLiteral         { value: number, position: number }
StringLiteral         { value: string, position: number }
BooleanLiteral        { value: boolean, position: number }
NullLiteral           { position: number }
ArrayExpression       { elements: Expression[], position: number }
ObjectExpression      { properties: Property[], position: number }

── Access (2) ────────────────────────────────
Identifier            { name: string, position: number }
MemberExpression      { object: Expression, property: string | Expression,
                        computed: boolean, optional: boolean, position: number }

── Operators (3) ─────────────────────────────
BinaryExpression      { op: string, left: Expression, right: Expression, position: number }
                      // op: "+", "-", "*", "/", "%", "**", "==", "!=", "<", ">", "<=", ">="
LogicalExpression     { op: string, left: Expression, right: Expression, position: number }
                      // op: "&&", "||", "??"
UnaryExpression       { op: string, argument: Expression, position: number }
                      // op: "!", "-"

── Control (1) ───────────────────────────────
ConditionalExpression { test: Expression, consequent: Expression,
                        alternate: Expression, position: number }

── Functions (2) ─────────────────────────────
ArrowFunction         { params: string[], body: Expression, position: number }
CallExpression        { callee: Expression, arguments: Expression[],
                        optional: boolean, position: number }

── Template (1) ──────────────────────────────
TemplateLiteral       { quasis: string[], expressions: Expression[], position: number }

── Pipe (1) ──────────────────────────────────
PipeExpression        { left: Expression, right: Expression, position: number }

── Spread (1) ────────────────────────────────
SpreadElement         { argument: Expression, position: number }
                      // NOTE: SpreadElement is defined but NOT used in v0.1
                      // Spread syntax is deferred to v0.2
```

### Property (used in ObjectExpression)

```
Property { key: string, value: Expression, position: number }
```

---

## Conformance Levels

Runtimes declare which conformance level they support:

### Level 1 — Core

Literals, arithmetic, comparison, logic, ternary, property access, function calls.

Includes:
- All literal types (number, string, boolean, null, array, object)
- All arithmetic operators (+, -, *, /, %, **)
- All comparison operators (==, !=, <, >, <=, >=)
- Logical operators (&&, ||, !)
- Ternary conditional (? :)
- Property access (dot notation, bracket notation)
- Array indexing
- All global built-in functions (math, type)
- Custom function registration

### Level 2 — Collections

Everything in Level 1, plus:
- Arrow functions (single and multi-param)
- All collection methods (map, filter, reduce, find, some, every, flatMap, sort, reverse)
- `.length` property on arrays
- String methods (len, upper, lower, trim, startsWith, endsWith, contains, split, replace, slice)
- Object methods (keys, values)
- Template literals

### Level 3 — Advanced

Everything in Level 2, plus:
- Pipe operator (`|>`)
- Optional chaining (`?.`)
- Nullish coalescing (`??`)

---

## Architecture

```
┌────────────────────────────┐
│   EBNF Grammar Spec        │  <- formal syntax definition (this document)
└────────────┬───────────────┘
             │ implemented as
┌────────────▼───────────────┐
│   Pratt Parser (per lang)  │  <- ~300-500 lines each, zero deps
└────────────┬───────────────┘
             │ produces
┌────────────▼───────────────┐
│  Language-idiomatic AST    │  <- 17 node types, internal to each runtime
└────────────┬───────────────┘
             │ walks
┌────────────▼───────────────┐
│     Tree-walk Evaluator    │  <- recursive eval, ~500 lines each
└────────────┬───────────────┘
             │ verified by
┌────────────▼───────────────┐
│  Conformance Test Suite    │  <- expression + context -> expected result
│  (YAML, 120+ cases)        │
└────────────────────────────┘
```

The AST is an **implementation detail** per runtime. The contract is:

> Given this expression and this context, produce this result.

Each runtime has its **own parser + evaluator** — not a shared WASM blob, not a shared protobuf schema. This keeps each implementation idiomatic and dependency-free.

---

## Parser Strategy

| Approach | JS | Python | Go | Dependency | Verdict |
|---|---|---|---|---|---|
| **ANTLR** | yes | yes | yes | Heavy runtime per language | Too heavy for an expression lang |
| **PEG generators** | Peggy | Parsimonious/Lark | Pigeon | Medium, different APIs | Possible but fragmented |
| **Hand-written Pratt** | yes | yes | yes | **Zero** | **Winner** |

Pratt is superior for expression languages because:
1. Operator precedence is a data table, not grammar rules — easy to extend
2. The algorithm is ~100 lines in any language — trivially portable
3. No codegen step, no grammar file to maintain, no build dependency
4. The same `expression(rbp)` function looks nearly identical in JS, Python, Go

---

## Deliverables

| # | Deliverable | Status |
|---|---|---|
| 1 | Grammar spec (EBNF) — this document | ✓ |
| 2 | Conformance test suite (YAML, 120+ cases) | In progress |
| 3 | JavaScript runtime (`@xpr-lang/xpr` on npm) | In progress |
| 4 | Python runtime (`xpr-lang` on PyPI) | Planned |
| 5 | Go runtime (`github.com/xpr-lang/xpr-go`) | Planned |
| 6 | Playground (web) | Planned |

---

## Future (v0.2+)

Features explicitly deferred from v0.1:

| Feature | Reason for Deferral |
|---------|---------------------|
| **Date/time functions** | Timezone handling, format parsing, locale — enormous complexity |
| **Spread operator** (`...`) | Three distinct grammar contexts (array, object, function args) — each is a separate parser rule |
| **`let` bindings** | Requires scope management, changes expression-only model |
| **Destructuring** | Complex grammar, multiple forms (array, object, nested) |
| **Pattern matching** | Requires type system extensions |
| **Async expressions** | Fundamentally changes evaluation model |
| **Custom operator overloading** | Requires type system |
| **Type annotations** | Requires type system |
| **Regex literals** | `/pattern/flags` syntax — adds tokenizer complexity |
| **Negative array indexing** | `items[-1]` — requires special-casing in evaluator |
