# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```sh
gleam build          # Build the project
gleam test           # Run all tests
gleam test -- --exact json_schema_test:string_schema_test  # Run a single test (gleeunit on Erlang)
gleam format         # Format code
```

## Architecture

This is a Gleam library that pairs JSON Schema generation with JSON decoding in a single API. A `JsonSchema(t)` value carries both a `SchemaNode` (for schema output) and a `decode.Decoder(t)` (for parsing JSON), keeping them in sync by construction.

**Key design pattern: continuation-passing for objects.** The `field`, `optional`, and `optional_or_null` functions use Gleam's `use` syntax with a continuation `fn(a) -> JsonSchema(b)`. They probe the continuation with a `coerce_nil()` FFI call to extract the full schema structure at build time, while building the real decoder separately. This is how the library collects all fields from a chain of `use` bindings into a single `ObjectNode`.

**Schema representation** (`SchemaNode`): An internal AST with variants for primitives (`StringNode`, `IntegerNode`, etc.), composites (`ArrayNode`, `NullableNode`, `ObjectNode`), and annotations (`DescriptionNode`). Converted to JSON via `node_to_pairs` which returns key-value pairs, allowing nodes like `NullableNode` and `DescriptionNode` to compose by modifying/appending pairs from their inner node.

**FFI files** (`src/json_schema_ffi.erl`, `src/json_schema_ffi.mjs`): Provide `coerce_nil()` which returns a nil/undefined value used to probe continuations. Both Erlang and JS targets must be maintained.

## Conventions

- The public API uses `js.` prefix by convention (users import as `import json_schema as js`)
- Tests assert both schema output (JSON string equality) and decode behavior (round-trip)
- The library targets both Erlang and JavaScript
