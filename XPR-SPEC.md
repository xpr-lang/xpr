# XPR — Cross-Language Expression Language

A sandboxed expression language with JS/Python-familiar syntax, designed for data pipeline transforms, with native interpreters in JavaScript, Python, and Go.

**Version**: 0.3.0  
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
- No variable assignment (`const`, `var`). Immutable `let` bindings ARE supported (see [v0.2 Features](#v02-features)).
- No loops (`for`, `while`). Collection operations are the only iteration.

---

## v0.2 Features

### `let` Bindings

`let` creates **immutable, scoped bindings**. It is NOT variable assignment — bindings cannot be re-assigned.

**Syntax**: `let <name> = <value>; <body>`

The semicolon separates the binding from the body expression. The body is the return value.

```javascript
let x = 1; x + 1              // 2
let x = 1; let y = x + 1; y  // 2 — chained bindings
let x = 1; let x = 2; x      // 2 — shadowing allowed
let f = (x) => x * 2; f(5)   // 10 — arrow function as value
```

**Scoping rules**:
- Each `let` binding is visible in the body expression that follows it
- Shadowing is allowed: a new `let x` in the body shadows the outer `x`
- Arrow functions capture bindings from their enclosing scope (closures)

**`let` is allowed everywhere an expression is valid**:
```javascript
// Top-level
let data = [1,2,3]; data |> map(x => x * 2)

// In ternary
x > 0 ? let a = x * 2; a : 0

// In array element
[let x = 1; x * 2, let y = 3; y]  // [2, 3]
```

**Error cases**:
- `let x = 1;` — trailing semicolon with no body → error: "Expected expression after ';'"
- `let = 1; 2` — missing identifier → parse error
- `let 123 = 1; 2` — non-identifier name → parse error

**Breaking change**: `let` is now a reserved keyword. Expressions using `let` as a variable name (e.g., context `{let: 5}` with expression `let + 1`) will fail to parse.

---

### Spread Operator

The spread operator (`...`) expands arrays or objects in-place.

**Array spread**:
```javascript
[...[1,2], 3, 4]              // [1,2,3,4]
[...[1,2], ...[3,4]]          // [1,2,3,4]
[0, ...items, 99]             // prepend/append with context variable
[...[]]                       // [] — spread empty array
```

**Object spread**:
```javascript
{...{"a":1}, "b":2}           // {"a":1,"b":2}
{...{"a":1}, ...{"b":2}}      // {"a":1,"b":2}
{...defaults, ...overrides}   // later keys override earlier keys
{...{}}                       // {} — spread empty object
```

**Error cases**:
- `[...null]` → error: "Cannot spread null"
- `[..."hello"]` → error: "Cannot spread string into array"
- `[...42]` → error: "Cannot spread non-array into array"
- `{...null}` → error: "Cannot spread null"
- `{...[1,2]}` → error: "Cannot spread array into object"
- `{...42}` → error: "Cannot spread non-object"

**Spread in function call arguments (v0.3)**:

```javascript
fn(...args)              // spread array as arguments
fn(a, ...rest)           // mix regular and spread arguments
fn(...a, b, ...c)        // multiple spreads with interspersed values
obj.method(...args)      // works with method calls
x |> fn(...rest)         // pipe prepends x, then spread expands: fn(x, rest[0], rest[1], ...)
```

**Error cases** (consistent with array/object spread):
```javascript
fn(...null)    // Error: Cannot spread null
fn(...42)      // Error: Cannot spread non-array
fn(..."hello") // Error: Cannot spread string into arguments
```

**NOT supported**: Spread in arrow function parameter definitions (`(...args) => ...` rest params are not supported).

---

### New Array Methods (v0.2)

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `includes` | `(val) → boolean` | `boolean` | Strict equality |
| `indexOf` | `(val) → number` | `number` | Returns `-1` if not found |
| `slice` | `(start[, end]) → array` | `array` | Same semantics as string `slice` |
| `join` | `(sep: string) → string` | `string` | Separator must be string |
| `concat` | `(other: array) → array` | `array` | Returns new array |
| `flat` | `() → array` | `array` | Flattens ONE level only |
| `unique` | `() → array` | `array` | Preserves first occurrence; identity comparison for objects |
| `zip` | `(other: array) → array` | `array of pairs` | Truncates to shortest length |
| `chunk` | `(size: number) → array` | `array of arrays` | Last chunk may be smaller; size must be > 0 |
| `groupBy` | `(fn: arrow) → object` | `object` | Keys are strings; **alphabetical key order** |

```javascript
[1,2,3].includes(2)                          // true
[1,2,3].indexOf(5)                           // -1
[1,2,3,4].slice(1,3)                         // [2,3]
[1,2,3].join(", ")                           // "1, 2, 3"
[1,2].concat([3,4])                          // [1,2,3,4]
[[1,2],[3,4]].flat()                         // [1,2,3,4]
[1,2,1,3].unique()                           // [1,2,3]
[1,2,3].zip([4,5,6])                         // [[1,4],[2,5],[3,6]]
[1,2,3,4].chunk(2)                           // [[1,2],[3,4]]
[1,2,3,4].groupBy(x => x % 2 == 0 ? "even" : "odd")  // {"even":[2,4],"odd":[1,3]}
```

---

### New String Methods (v0.2)

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `indexOf` | `(sub: string) → number` | `number` | Returns `-1` if not found |
| `repeat` | `(n: number) → string` | `string` | `n` must be non-negative integer |
| `trimStart` | `() → string` | `string` | Removes leading whitespace |
| `trimEnd` | `() → string` | `string` | Removes trailing whitespace |
| `charAt` | `(n: number) → string` | `string` | Out of bounds → `""` (empty string) |
| `padStart` | `(n[, ch]) → string` | `string` | `ch` defaults to `" "` |
| `padEnd` | `(n[, ch]) → string` | `string` | `ch` defaults to `" "` |

```javascript
"hello".indexOf("ll")    // 2
"hello".indexOf("xyz")   // -1
"ab".repeat(3)           // "ababab"
"  hello".trimStart()    // "hello"
"hello  ".trimEnd()      // "hello"
"hello".charAt(1)        // "e"
"hello".charAt(99)       // ""
"5".padStart(3, "0")     // "005"
"5".padEnd(3, ".")       // "5.."
```

---

### New Object Methods (v0.2)

| Method | Signature | Returns | Notes |
|--------|-----------|---------|-------|
| `entries` | `() → array` | `array of [key, value] pairs` | **Alphabetical key order** (required for cross-runtime conformance) |
| `has` | `(key: string) → boolean` | `boolean` | Checks if key exists at top level |

```javascript
{"c":3,"a":1,"b":2}.entries()  // [["a",1],["b",2],["c",3]]
{"a":1}.has("a")               // true
{"a":1}.has("b")               // false
```

---

### New Global Function: `range` (v0.2)

```javascript
range(end)              // range(5)       → [0,1,2,3,4]
range(start, end)       // range(2,5)     → [2,3,4]
range(start, end, step) // range(0,10,2)  → [0,2,4,6,8]
                        // range(5,0,-1)  → [5,4,3,2,1]
```

- `step` must be an integer — float step → error
- Negative step counts down: `range(5,0,-1)` → `[5,4,3,2,1]`
- Empty range if start ≥ end with positive step: `range(5,0)` → `[]`

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
items[-1]   // last element (equivalent to items[items.length - 1])
items[-2]   // second-to-last element
items[-0]   // first element (-0 === 0 in IEEE 754)
[][- 1]     // null — out of bounds on empty array
items[-99]  // null — out of bounds if abs(index) > length
```

Negative index `n` accesses `arr[arr.length + n]`. Returns `null` if the result is still out of bounds. Fractional negative indices are truncated to integer (e.g., `-1.7` → `-1`). Applies to arrays only — not strings.

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
- Arrow functions are **first-class values** — they can be passed as arguments to collection methods, used as pipe targets, and assigned to `let` bindings (e.g., `let f = (x) => x + 1; f(5)` → `6`)
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

### Date/Time Functions (v0.3)

Dates are represented as **epoch milliseconds** (number type). All operations are UTC. There is no separate `date` type.

| Function | Signature | Returns | Notes |
|----------|-----------|---------|-------|
| `now` | `() → number` | Epoch ms | Current UTC timestamp |
| `parseDate` | `(str, format?) → number` | Epoch ms | Default: ISO 8601. Custom format uses ICU tokens. |
| `formatDate` | `(date, format) → string` | Formatted string | ICU tokens only (see below) |
| `year` | `(date) → number` | Year (e.g., 2024) | UTC |
| `month` | `(date) → number` | Month 1–12 | 1-indexed |
| `day` | `(date) → number` | Day 1–31 | UTC |
| `hour` | `(date) → number` | Hour 0–23 | UTC |
| `minute` | `(date) → number` | Minute 0–59 | UTC |
| `second` | `(date) → number` | Second 0–59 | UTC |
| `millisecond` | `(date) → number` | Millisecond 0–999 | UTC |
| `dateAdd` | `(date, amount, unit) → number` | Epoch ms | Fractional amounts truncated. Month overflow: Jan 31 + 1 month = Mar 2/3. |
| `dateDiff` | `(date1, date2, unit) → number` | Signed integer | `date1 < date2` → positive. Truncated to integer. |

**ICU Format Tokens** (closed set — no others supported):

| Token | Meaning | Example |
|-------|---------|---------|
| `yyyy` | 4-digit year | `2024` |
| `MM` | 2-digit month (01–12) | `06` |
| `dd` | 2-digit day (01–31) | `15` |
| `HH` | 2-digit hour 24h (00–23) | `10` |
| `mm` | 2-digit minute (00–59) | `30` |
| `ss` | 2-digit second (00–59) | `45` |
| `SSS` | 3-digit millisecond (000–999) | `123` |

All other characters in the format string are treated as literals.

**Units for `dateAdd`/`dateDiff`**: `"years"`, `"months"`, `"days"`, `"hours"`, `"minutes"`, `"seconds"`, `"milliseconds"`

```javascript
now()                                              // e.g. 1710000000000
parseDate("2024-01-15T12:00:00Z")                  // 1705320000000
parseDate("15/01/2024", "dd/MM/yyyy")              // 1705276800000
formatDate(0, "yyyy-MM-dd")                        // "1970-01-01"
formatDate(parseDate("2024-06-15T10:30:45Z"), "HH:mm:ss")  // "10:30:45"
year(parseDate("2024-06-15T10:30:00Z"))            // 2024
month(parseDate("2024-06-15T10:30:00Z"))           // 6
day(parseDate("2024-06-15T10:30:00Z"))             // 15
dateAdd(parseDate("2024-01-31T00:00:00Z"), 1, "months")    // epoch ms for 2024-03-02
dateDiff(parseDate("2024-01-01T00:00:00Z"), parseDate("2024-01-08T00:00:00Z"), "days")  // 7
dateDiff(parseDate("2024-01-08T00:00:00Z"), parseDate("2024-01-01T00:00:00Z"), "days")  // -7
```

**Error cases**:
```javascript
parseDate(42)                    // Error: Type error — expected string
parseDate("not-a-date")          // Error: invalid date string
year(null)                       // Error: Type error — expected number
dateAdd(now(), 1, "weeks")       // Error: invalid unit "weeks"
```

### Regex Functions (v0.3)

Function-based regex using **RE2 flavor**. No literal syntax (`/pattern/` is not supported). Inline flags via RE2 syntax (e.g., `(?i)` for case-insensitive).

**RE2 constraint**: No lookahead, lookbehind, backreferences, or atomic groups.

| Function | Signature | Returns | Notes |
|----------|-----------|---------|-------|
| `matches` | `(str, pattern) → boolean` | `boolean` | Searches for pattern anywhere in string |
| `match` | `(str, pattern) → string \| null` | First matched substring or `null` | Returns string, not a match object |
| `matchAll` | `(str, pattern) → array` | Array of matched strings | Non-overlapping. Empty array if no matches. |
| `replacePattern` | `(str, pattern, replacement) → string` | New string | Replaces ALL matches. `$1`/`$2` for group references. |

```javascript
matches("hello world", "\\d+")                              // false
matches("hello 42", "\\d+")                                 // true
matches("Hello", "(?i)hello")                               // true
match("order-123", "\\d+")                                  // "123"
match("no digits", "\\d+")                                  // null
matchAll("a1b2c3", "\\d+")                                  // ["1", "2", "3"]
matchAll("no digits", "\\d+")                               // []
replacePattern("hello world", "o", "0")                     // "hell0 w0rld"
replacePattern("2024-01-15", "(\\d{4})-(\\d{2})-(\\d{2})", "$3/$2/$1")  // "15/01/2024"
```

**Error cases**:
```javascript
matches(42, "\\d+")          // Error: Type error — expected string
matches("test", "[invalid")  // Error: invalid regex pattern
match(null, "x")             // Error: Type error — expected string
```

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

Formal grammar for XPR v0.2. Uses standard EBNF notation:
- `*` = zero or more
- `+` = one or more
- `?` = optional
- `|` = alternation
- `( )` = grouping
- `[ ]` = character class

```ebnf
(* Top-level — may be a let binding or any expression *)
Expression     ::= LetExpression
               | PipeExpr

(* Let bindings — immutable, scoped *)
LetExpression  ::= "let" Identifier "=" Expression ";" Expression

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

ArgList        ::= ArgElement ( "," ArgElement )*

ArgElement     ::= "..." Expression                (* spread argument *)
               | Expression                        (* regular argument *)

(* Literals — arrays and objects support spread elements *)
ArrayLiteral   ::= "[" ( ArrayElement ( "," ArrayElement )* )? "]"

ArrayElement   ::= "..." Expression                (* spread *)
               | Expression                        (* regular element *)

ObjectLiteral  ::= "{" ( ObjectEntry ( "," ObjectEntry )* )? "}"

ObjectEntry    ::= "..." Expression                (* spread *)
               | Property                          (* regular property *)

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

19 node types. ESTree-influenced, simplified for expression-only context.

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

── Bindings (1) ──────────────────────────────
LetExpression         { name: string, value: Expression, body: Expression, position: number }

── Spread (1) ────────────────────────────────
SpreadElement         { argument: Expression, position: number }
                      // Used in ArrayExpression.elements, ObjectExpression.properties, and CallExpression.arguments
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
| 2 | Conformance test suite (YAML, 250+ cases) | ✓ |
| 3 | JavaScript runtime (`@xpr-lang/xpr` on npm) v0.3.0 | ✓ |
| 4 | Python runtime v0.3.0 | ✓ |
| 5 | Go runtime (`github.com/xpr-lang/xpr-go`) v0.3.0 | ✓ |
| 6 | Playground (web, CodeMirror 6) | ✓ |

---

## Future (v0.4+)

Features explicitly deferred from v0.3:

| Feature | Reason for Deferral |
|---------|---------------------|
| **Destructuring** | Complex grammar, multiple forms (array, object, nested) |
| **Pattern matching** | Requires type system extensions |
| **Async expressions** | Fundamentally changes evaluation model |
| **Custom operator overloading** | Requires type system |
| **Type annotations** | Requires type system |
| **Regex literals** | `/pattern/flags` syntax — adds tokenizer complexity. Function-based regex (`matches`, `match`, etc.) is supported in v0.3. |
| **Timezone-aware dates** | IANA timezone database, DST handling — enormous complexity. UTC-only date functions are supported in v0.3. |
