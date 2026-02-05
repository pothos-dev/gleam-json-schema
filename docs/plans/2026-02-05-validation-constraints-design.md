# String and Number Validation Constraints

## Summary

Add 8 pipeable constraint functions that both emit JSON Schema keywords and validate during decode. Drop `format` since we won't validate it.

## API

### String constraints

```gleam
js.string() |> js.min_length(3)           // {"type":"string","minLength":3}
js.string() |> js.max_length(100)         // {"type":"string","maxLength":100}
js.string() |> js.pattern("^[a-z]+$")     // {"type":"string","pattern":"^[a-z]+$"}
```

### Number constraints (Float params, work on both Int and Float schemas)

```gleam
js.integer() |> js.minimum(0.0)                // {"type":"integer","minimum":0}
js.number()  |> js.maximum(100.0)              // {"type":"number","maximum":100}
js.integer() |> js.exclusive_minimum(0.0)      // {"type":"integer","exclusiveMinimum":0}
js.number()  |> js.exclusive_maximum(100.0)    // {"type":"number","exclusiveMaximum":100}
js.number()  |> js.multiple_of(0.5)            // {"type":"number","multipleOf":0.5}
```

## Internal representation

One `SchemaNode` variant per constraint, wrapping an inner node (same pattern as `DescriptionNode`):

```
MinLengthNode(inner: SchemaNode, value: Int)
MaxLengthNode(inner: SchemaNode, value: Int)
PatternNode(inner: SchemaNode, pattern: String)
MinimumNode(inner: SchemaNode, value: Float)
MaximumNode(inner: SchemaNode, value: Float)
ExclusiveMinimumNode(inner: SchemaNode, value: Float)
ExclusiveMaximumNode(inner: SchemaNode, value: Float)
MultipleOfNode(inner: SchemaNode, value: Float)
```

Each variant:
- `node_to_pairs`: delegates to inner, appends its keyword (e.g., `#("minLength", json.int(value))`)
- `get_type_name`: delegates to inner (passthrough)

## Decode validation

Each constraint wraps the inner decoder with `decode.then`:

- **min_length**: `string.length(s) >= min`, failure message `"string with minLength N"`
- **max_length**: `string.length(s) <= max`, failure message `"string with maxLength N"`
- **pattern**: `regex.from_string(pat)` then `regex.check`, failure message `"string matching pattern /P/"`
- **minimum**: `val >=. min`, failure `">= N"`
- **maximum**: `val <=. max`, failure `"<= N"`
- **exclusive_minimum**: `val >. min`, failure `"> N"`
- **exclusive_maximum**: `val <. max`, failure `"< N"`
- **multiple_of**: check remainder is zero, failure `"multiple of N"`

For Int schemas: the decoder receives `Int`, converts to `Float` via `int.to_float` for comparison, returns the original `Int` on success.

## Type signatures

```gleam
pub fn min_length(schema: JsonSchema(String), min: Int) -> JsonSchema(String)
pub fn max_length(schema: JsonSchema(String), max: Int) -> JsonSchema(String)
pub fn pattern(schema: JsonSchema(String), regex: String) -> JsonSchema(String)
pub fn minimum(schema: JsonSchema(a), min: Float) -> JsonSchema(a)
pub fn maximum(schema: JsonSchema(a), max: Float) -> JsonSchema(a)
pub fn exclusive_minimum(schema: JsonSchema(a), min: Float) -> JsonSchema(a)
pub fn exclusive_maximum(schema: JsonSchema(a), max: Float) -> JsonSchema(a)
pub fn multiple_of(schema: JsonSchema(a), value: Float) -> JsonSchema(a)
```

Number functions are generic (`JsonSchema(a)`) since Gleam can't express `Int | Float`. Type safety relies on users only having `JsonSchema(Int)` or `JsonSchema(Float)` from the primitive constructors.

## Dependencies

- `gleam/regex` — needed for `pattern` validation
- `gleam/int` — needed for `int.to_float` in number validation
- `gleam/float` — needed for float comparisons and modulo

## What's NOT included

- `format` — omitted since we don't validate it
- Array constraints (`minItems`, `maxItems`, `uniqueItems`) — future work
- Object constraints (`additionalProperties`, etc.) — future work
