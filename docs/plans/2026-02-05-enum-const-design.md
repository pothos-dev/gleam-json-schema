# Enum and Const Support

## Summary

Add `enum` and `const` JSON Schema keywords for string values, with both plain (decode to `String`) and mapped (decode to custom Gleam type) variants.

## SchemaNode Changes

Two new variants:

```
EnumNode(values: List(String))
ConstNode(value: String)
```

`node_to_pairs` produces:
- `EnumNode(["red", "green"])` -> `{"type": "string", "enum": ["red", "green"]}`
- `ConstNode("active")` -> `{"type": "string", "const": "active"}`

`get_type_name` returns `Ok("string")` for both, so they compose with `NullableNode` and `DescriptionNode` without special cases.

## Public API

```gleam
// Decode to String
js.enum(["red", "green", "blue"])
js.constant("active")

// Decode to custom type
js.enum_map([#("red", Red), #("green", Green), #("blue", Blue)])
js.constant_map("active", Active)
```

All four produce the same schema shapes. The `_map` variants differ only in the decoder.

## Decoders

Use `decode.one_of` with one branch per allowed value. Each branch decodes a string and checks for an exact match.

- `enum(values)`: `one_of` over values, each succeeding with the matched string.
- `enum_map(variants)`: `one_of` over variants, each succeeding with the mapped Gleam value.
- `constant(value)`: same as `enum([value])` internally.
- `constant_map(value, mapped)`: same as `enum_map([#(value, mapped)])` internally.

Unmatched values produce a standard `decode` error (no custom error types).

## Tests

**Schema output:**
- `enum_schema_test` -- basic enum output
- `enum_map_schema_test` -- same schema as plain enum
- `const_schema_test` -- basic const output
- `const_map_schema_test` -- same schema as plain const
- `enum_with_describe_test` -- description composes
- `nullable_enum_test` -- nullable composes

**Decode:**
- `decode_enum_valid_test` -- matching value succeeds
- `decode_enum_invalid_test` -- non-matching returns error
- `decode_enum_map_test` -- maps to Gleam variant
- `decode_const_test` -- matching succeeds
- `decode_const_invalid_test` -- non-matching returns error
- `decode_const_map_test` -- maps to Gleam value

**Integration:**
- Enum/const field inside an object schema using `field`/`optional`
