---
name: jsonata-syntax
description: Learn to PROGRAM in JSONata — the JSON query-and-transform language. Use when writing or debugging a JSONata expression: an actions.json workflow/projection slot, a Kestra/Stedi/Node-RED mapping, an `$map`/`$filter`/`$reduce` transform, a `$string`/`$substring`/`$split` string op, or when an expression returns nothing, the wrong shape, or a parse error ("Expected ...", "did not match"). Teaches the mental model, not just a function list.
---

# JSONata: how to actually program in it

JSONata is a small, purely-functional language for **querying and transforming JSON**. You write one expression; it evaluates against an input JSON value and returns a new JSON value. There are no statements that "do" things and no mutation — **everything is an expression that returns a value**, and you compose bigger transforms out of smaller ones.

This skill teaches the *mental model* so you can write JSONata from understanding instead of guessing. It is grounded against the engine actually vendored in this project (`extensions/chrome-overlay-runtime/src/agent/vendor/jsonata.mjs`) — every example below was run through it. Verify your own expressions the same way (see **Test it, don't guess it** at the end).

Official docs: https://docs.jsonata.org/ · Try expressions live: https://try.jsonata.org/

---

## The one idea everything hangs on: sequences

This is the concept that, once you get it, explains 80% of JSONata's "surprises." **A path expression does not return a value — it returns a *sequence* of values.** A sequence is JSONata's internal flat stream of results. Four rules govern how a sequence turns into your JSON output, and they are the root of nearly every gotcha:

1. **Empty sequence → nothing.** No match produces "nothing" (JSONata calls it *undefined*). It disappears from the output — it is not `null`, not `[]`. `Address.Zip` on a doc with no Zip yields *nothing*, and any expression built on nothing tends to also yield nothing.
2. **Singleton sequence → the bare value.** A sequence of exactly one item **collapses to that item, with no array around it.** `$filter([0,0,5,0], fn>0)` returns `5`, **not** `[5]`. This bites constantly.
3. **Multi-value sequence → a JSON array.** Two or more results render as an array.
4. **Sub-sequences flatten.** A sequence never nests; child sequences are pulled up into the parent. `Phone.number` gathers *all* numbers into one flat array, not an array-of-arrays.

Hold this the whole time: **you are steering a flat stream of values, and a stream of one silently becomes a scalar.** When an expression "should return an array" but downstream code chokes on a scalar, rule 2 is why.

---

## Stage 1 — Navigate: location paths

Paths are the spine. Dot-separated field names dig into objects; the result is a sequence.

| Expression | On `{"Surname":"Smith","Address":{"City":"Winchester"}}` | Note |
|---|---|---|
| `Surname` | `"Smith"` | field access |
| `Address.City` | `"Winchester"` | nested |
| `Other.` + `` `'Over 18 ?'` `` | `true` | quote field names with spaces/punctuation using backticks |
| `**.Postcode` | every `Postcode` at any depth | `**` = descendant wildcard |
| `Address.*` | all values of `Address` | `*` = one-level wildcard |

The context is implicit — you write `City`, not `$.City` (though `$` means "the current context value" when you need it explicitly, e.g. at the top or inside a function).

## Stage 2 — Index and filter: `[ ]`

Brackets after a path do one of two things depending on what's inside:

- **Integer → positional index** (0-based, negatives from the end): `Phone[0]`, `Phone[-1]` (last).
- **Boolean expression → predicate filter**: `Phone[type='mobile']` keeps items where the expression is true.

```
Phone[0].number          → "0203 544 1234"    (first phone's number)
Phone[type='mobile'].number                    (numbers of mobile phones only)
(Phone.number)[0]        → "0203 544 1234"    (ALL numbers, then the first)
```

**Precedence trap (write this on your hand): the filter binds TIGHTER than the dot.** So `books.authors[0]` means "the first author *of each book*", not "the first author overall." To get the overall first, force the order with parentheses: `(books.authors)[0]`. This one surprise is behind a huge fraction of "my index returned the wrong thing."

## Stage 3 — Combine: operators

- **String concat:** `&` — `FirstName & ' ' & Surname` → `"Fred Smith"`. (`+` is numeric only; `'a' + 'b'` errors.)
- **Arithmetic:** `+ - * / %`.
- **Comparison:** `= != < <= > >=`, and `in` (membership): `"mobile" in Phone.type`.
- **Boolean:** `and`, `or` (the words, not `&&`/`||`).

## Stage 4 — Reshape: build new JSON

You construct output by writing object/array literals whose values are expressions:

- **Object constructor:** `{ 'name': FirstName & ' ' & Surname, 'city': Address.City }`.
- **Grouping object constructor** (the powerful one): `Phone{type: number}` groups by the key expression → `{"home":"1","mobile":"2"}`. Applied to a sequence, `expr{ keyExpr: valueExpr }` builds an object keyed by `keyExpr`.
- **Array constructor:** `[ a, b, c ]` — and crucially, `[ ... ]` **forces an array**, defeating singleton-collapse (rule 2). `[$filter([0,0,5,0], fn>0)]` → `[5]`.

---

## Stage 5 — Program: this is where it becomes a language

### Blocks and variable bindings

Wrap several steps in parentheses, separate with `;`, and **the value of the block is its last expression**. Bind variables with `:=` (variables are `$`-prefixed and scoped to their block):

```
(
  $p := Product.Price;
  $q := Product.Quantity;
  $p * $q
)
```

This block shape — `( $x := ...; $y := ...; finalExpr )` — is your main tool for readable multi-step logic, and it works *inside* a larger expression. **Watch the syntax carefully:** the `;`-separated statements must be inside `( )`. A `( $x := f(); expr )` embedded as one arm of a ternary is fine; a bare `a ? (...;...) : b` is fine; but you cannot scatter `:=` statements at the top level of an expression slot without the enclosing parens. (Getting this wrong yields `Expected "}" , got ";"` — the classic first-timer parse error.)

### Conditionals

- Ternary: `predicate ? whenTrue : whenFalse`.
- Elvis (default-if-falsy): `a ?: b`.
- Coalesce (default-if-undefined): `a ?? b`.

### Functions are values

Define an anonymous function and bind it like anything else:

```
$volume := function($l, $w, $h){ $l * $w * $h };
$volume(2, 3, 4)     → 24
```

Functions are **first-class**: pass them to other functions, return them, store them. They **close over** their defining environment. Recursion works (reference the function by its variable name), and tail calls are optimized to loops so deep recursion won't overflow.

### The higher-order trio (the heart of transforms)

Every array reshape is usually one of these. The callback receives `($value, $index, $array)` — take only the params you need:

- **`$map(arr, fn)`** — transform each: `$map([1,2,3], function($v){ $v * 2 })` → `[2,4,6]`.
- **`$filter(arr, fn)`** — keep where true: `$filter(Products, function($v){ $v.price > 10 })`.
- **`$reduce(arr, fn, init)`** — fold to one value: `$reduce([1,2,3,4], function($acc,$v){ $acc + $v }, 0)` → `10`.
- **`$sift(obj, fn)`** — keep object key/values where the predicate holds (filter *object fields*).
- **`$each(obj, fn)`** — map over an object's key/value pairs.

### Chaining with `~>`

`value ~> $f ~> $g` is `$g($f(value))` — reads left-to-right like a pipeline: `'hello' ~> $uppercase` → `"HELLO"`. And `$h := $f ~> $g` *composes* a new function. Partial application with `?` makes a smaller function: `$first5 := $substring(?, 0, 5)`.

---

## The gotchas that will actually bite you

These are the ones worth memorizing — each is a real, reproducible surprise (all verified against the vendored engine):

1. **Singleton collapse.** One result is a bare scalar, not a one-element array. If a step *might* return a single item and you need an array, wrap it: `[ expr ]`, or preserve arrays through a map with `$append([], $map(...))`. Symptom: downstream `$count`/index/`$map` behaves as if it got a scalar.

2. **Absence is silently false in comparisons — use `$exists`, never `= null`.** A missing path compared to anything is false, *and so is its negation*:
   ```
   nested.a.z = null   → false     (even though z is missing!)
   nested.a.z != null  → false     (both false — you cannot detect absence this way)
   $exists(nested.a.z) → false     ✓ the correct absence test
   ```
   Guard every `when`/predicate on an optional path with `$exists(...)`, or your fallback branch is skipped exactly when it's needed. This is the single highest-value rule for authoring robust expressions.

3. **`$map`/`$filter` can collapse empty or singleton results** the same way (rule 2). Preserve arrays deliberately with `$append([], ...)` and guard empties with `$count(x) > 0 ? ... : []`.

4. **Filter binds tighter than dot** (Stage 2) — parenthesize when you mean "of the whole result."

5. **`&` for strings, `+` for numbers** — mixing them errors; concatenate with `&`.

6. **Field names with spaces/punctuation need backticks** — `` `'Over 18 ?'` ``, or the parser reads them as operators.

7. **Boolean ops on undefined are inconsistent** — combine `or`/`and` with possibly-missing operands only after `$exists`-guarding them.

8. **Copy-paste typography breaks real expressions.** Smart quotes (`“ ”`), curly apostrophes (`’`), and invisible pasted characters are not JSONata syntax. If an expression looks right but the parser complains near a string, retype the quotes as plain ASCII and test the smallest expression first.

---

## Function cheat-sheet (grounded in the vendored engine)

Reach for the built-in before writing logic. All `$`-prefixed.

- **String:** `$string`, `$length`, `$substring(str, start, len)`, `$substringBefore`, `$substringAfter`, `$uppercase`, `$lowercase`, `$trim`, `$pad`, `$contains(str, sub)`, `$split(str, sep)`, `$join(arr, sep)`, `$replace(str, pattern, repl)` (pattern can be a `/regex/`), `$match`, `$base64encode/decode`.
- **Numeric:** `$number`, `$abs`, `$floor`, `$ceil`, `$round`, `$power`, `$sqrt`, `$min`, `$max`, `$sum`, `$average`, `$formatNumber`.
- **Array:** `$count`, `$append(a, b)`, `$sort(arr, fn)`, `$reverse`, `$shuffle`, `$distinct`, `$zip`, `$map`, `$filter`, `$reduce`.
- **Object:** `$keys`, `$lookup(obj, key)`, `$spread`, `$merge`, `$sift`, `$each`, `$type`, `$exists`.
- **Aggregation over paths** works directly: `$sum(Order.total)`, `$max(Phone.number ~> $number)`.

When unsure a function exists or behaves as you think in *this* engine, test it (below) — don't trust memory across JSONata versions.

---

## A worked transform (composition, end-to-end)

Given `{"orders":[{"items":[{"p":10,"q":2},{"p":5,"q":4}]},{"items":[{"p":3,"q":1}]}]}`, produce per-order totals and a grand total:

```
(
  $orderTotals := $map(orders, function($o) {
    $sum($map($o.items, function($i){ $i.p * $i.q }))
  });
  {
    'perOrder': [$orderTotals],
    'grand': $sum($orderTotals)
  }
)
```

→ `{"perOrder":[40,3],"grand":43}`. Note the `[$orderTotals]` — without the array-forcing brackets, a single-order input would collapse `perOrder` to a scalar (gotcha #1). This is the whole skill in one expression: paths gather, `$map` transforms, `$sum` folds, a block binds intermediate steps, and array-forcing defeats singleton-collapse.

---

## Advanced composition patterns

Use these when a one-line path stops being honest. They are compact, but each one combines several core ideas.

### Recursive flatten: blocks + recursion + `$each` + `~> $merge`

Given nested JSON, produce one flat object with dotted keys:

```
(
  $flatten := function($o, $prefix) {
    $each($o, function($v, $k) {(
      $name := $prefix ? $prefix & '.' & $k : $k;
      $type($v) = 'object' ? $flatten($v, $name) : { $name: $v }
    )}) ~> $merge()
  };
  $flatten($, '')
)
```

On `{"customer":{"name":"Ada","address":{"city":"London","zip":"SW1"}},"active":true}`:

```
{
  "customer.name": "Ada",
  "customer.address.city": "London",
  "customer.address.zip": "SW1",
  "active": true
}
```

Read it in layers: `$each` walks an object, the block binds the dotted name, the ternary recurses only for object values, each leaf returns a one-entry object, and `~> $merge()` combines the array of one-entry objects into one object.

### Group, then reshape

Object constructors can group a sequence by a computed key. Then `$each` can turn the grouped object back into an array:

```
(
  $groups := orders{date: sku[]};
  $each($groups, function($skus, $date) {
    { 'date': $date, 'skus': [$skus] }
  })
)
```

On three orders with two dates this returns:

```
[
  { "date": "2026-07-01", "skus": ["A", "B"] },
  { "date": "2026-07-02", "skus": ["C"] }
]
```

The `sku[]` and `[$skus]` are intentional shape protection: a date with one SKU should still produce a `skus` array.

---

## Parse errors: read the punctuation

JSONata parse errors are often terse, but the fix is usually in the punctuation.

- **`Expected "}" got ":"` or similar inside a callback** usually means you are building an object in a place where the parser did not see a complete object expression. Reduce the callback to one returned object, then add fields back one at a time.
- **`Expected "}" got ";"`** usually means you put block statements where an object constructor was expected, or you forgot that semicolons belong inside `( ... )` blocks.
- **`{ ... }` constructs an object.** It expects key/value pairs: `{ 'name': value }`.
- **`( ... )` creates a grouped expression or block.** It may contain `:=` bindings and `;` separators: `( $x := 1; $x + 1 )`.
- **Quote non-identifier object keys.** Keys like `"8"`, `"field-id"`, or `"Over 18 ?"` must be quoted or backticked in the right context.

Safe callback pattern:

```
$map(fields, function($f) {
  { '8': { 'value': $f.value } }
})
```

When stuck, do not keep editing the full expression. Test the smallest syntactic unit: first the object literal, then the callback, then the `$map`.

---

## In an actions.json map specifically

Map workflows and state projections embed JSONata in **whole-string `{% ... %}` slots** (a `repeat`, an `output`, a projection `expression`). Constraints that matter there:

- **Whole-string only.** The entire slot is one JSONata expression: `"{% input.chars - 1 %}"`. Don't try to interpolate around it.
- **Context bindings the engine injects:** `input` (validated action args), `steps.<id>.output` (a prior step's result), and `item`/`index` inside a `for_each` loop.
- **`$exists` on every optional `steps.*.output.*`** — a step may not have run, and (gotcha #2) `= null` won't catch it.
- **Preserve arrays** in projections with `$append([], $map(...))` and guard empties with `$count(x) > 0 ? ... : []`, so a one-item page doesn't collapse your list to a scalar.
- The authoring skill `write-actions-json` covers the surrounding contract (steps, `when`, `retry_until`); this skill covers the expression language inside those slots.

---

## Host boundaries: JSONata plus local rules

Many products embed JSONata but add their own root variables, delimiters, functions, or restrictions. First separate **vanilla JSONata** from **host contract**.

- **actions.json maps:** whole-string `{% ... %}` slots with injected `input`, `steps.<id>.output`, and loop `item`/`index`.
- **AWS Step Functions:** expressions are also delimited with `{% ... %}`, but data comes through `$states.input`, `$states.result`, and related host variables. A JSONPath habit like `$.foo` is usually the wrong mental model there.
- **Node-RED:** JSONata may include host functions such as `$flowContext()` and `$globalContext()`. Those are not portable vanilla JSONata.
- **Stedi, Truto, Kestra, and other mapping products:** check the platform's root object and custom functions before copying examples across hosts.

Debug host issues in two passes: first prove the expression in a vanilla JSONata engine with an equivalent input object; then add the host-specific variables or delimiters back.

---

## Test it, don't guess it

The fastest way to be right is to evaluate the expression against sample input before shipping it. Two ways:

- **Live, zero-setup:** paste into https://try.jsonata.org/ with your sample JSON.
- **Against this project's exact engine** (so it's true for our maps):
  ```js
  // node — grounds against the vendored engine the runtime actually uses
  import jsonata from '/absolute/path/to/actions.json.dev/extensions/chrome-overlay-runtime/src/agent/vendor/jsonata.mjs';
  const r = await jsonata("$map(nums, function($v){$v*2})").evaluate({ nums: [1,2,3] });
  console.log(JSON.stringify(r)); // [2,4,6]
  ```
  Feed it the real fixture (the paragraph text, the extracted DOM records, the `input` shape) and check the output shape, not just that it ran. A parse error means the *syntax* is wrong (usually a block/`;`/paren issue — Stage 5); a wrong-but-valid result means the *sequence logic* is wrong (usually gotcha #1 or #2).

**Sources:** JSONata official docs — [processing model](https://docs.jsonata.org/processing), [path operators](https://docs.jsonata.org/path-operators), [predicates](https://docs.jsonata.org/predicate), [programming constructs](https://docs.jsonata.org/programming), [higher-order functions](https://docs.jsonata.org/higher-order-functions), and the [official tutorial](https://github.com/jsonata-js/jsonata/blob/master/tutorial.md). Additional teaching examples were cross-checked from community war stories and worked transforms: [recursive flatten](https://stackoverflow.com/questions/60817650/how-to-flatten-nested-object-to-single-depth-object-with-jsonata), [group values](https://stackoverflow.com/questions/50697006/group-values-in-jsonata), [Step Functions singleton array shape](https://stackoverflow.com/questions/79291760/step-functions-jsonata-mapiterator-bug), [Step Functions host variables](https://stackoverflow.com/questions/79304259/how-do-i-evaluate-jsonata-expression-in-key-in-json), and Node-RED/Home Assistant parse-error threads. All examples verified against `extensions/chrome-overlay-runtime/src/agent/vendor/jsonata.mjs`.
