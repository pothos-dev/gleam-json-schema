# json_schema

A Gleam library for JSON Schema generation and decoding. Define a schema once, then use it to both generate a JSON Schema string and decode JSON values into typed Gleam data.

[![Package Version](https://img.shields.io/hexpm/v/json_schema)](https://hex.pm/packages/json_schema)
[![Hex Docs](https://img.shields.io/badge/hex-docs-ffaff3)](https://hexdocs.pm/json_schema/)

```sh
gleam add json_schema@1
```

## How it works

`JsonSchema(t)` is an opaque type that pairs a JSON Schema definition with a decoder. When you build a schema using the builder API, you get a value that can:

- **Generate** a JSON Schema string via `to_string`
- **Decode** a JSON value into a typed Gleam result via `decode`

The schema and decoder are always in sync -- if you say a field is a string, the decoder knows to decode a string.

## Quick start

```gleam
import json_schema as js

pub type User {
  User(name: String, age: Int)
}

fn user_schema() {
  use name <- js.field("name", js.string())
  use age <- js.field("age", js.integer())
  js.success(User(name:, age:))
}

pub fn main() {
  let schema = user_schema()

  // Generate JSON Schema
  js.to_string(schema)
  // -> {"type":"object","properties":{"name":{"type":"string"},"age":{"type":"integer"}},"required":["name","age"]}

  // Decode JSON values
  js.decode(schema, from: "{\"name\":\"Alice\",\"age\":30}")
  // -> Ok(User(name: "Alice", age: 30))
}
```

## Full example

A more realistic schema with nested objects, arrays, optional fields, nullable fields, and descriptions:

```gleam
import gleam/option.{type Option}
import json_schema as js

pub type Address {
  Address(street: String, city: String, zip: Option(String))
}

pub type Tag {
  Tag(key: String, value: String)
}

pub type Company {
  Company(
    name: String,
    founded_year: Int,
    public: Bool,
    rating: Option(Float),
    address: Address,
    tags: List(Tag),
    website: Option(String),
    phone: Option(String),
  )
}

fn address_schema() {
  use street <- js.field("street", js.string() |> js.describe("Street address"))
  use city <- js.field("city", js.string())
  use zip <- js.optional("zip", js.string() |> js.describe("ZIP or postal code"))
  js.success(Address(street:, city:, zip:))
}

fn tag_schema() {
  use key <- js.field("key", js.string())
  use value <- js.field("value", js.string())
  js.success(Tag(key:, value:))
}

fn company_schema() {
  use name <- js.field("name", js.string() |> js.describe("Legal company name"))
  use founded_year <- js.field("founded_year", js.integer() |> js.describe("Year the company was founded"))
  use public <- js.field("public", js.boolean() |> js.describe("Whether publicly traded"))
  use rating <- js.optional_or_null("rating", js.number() |> js.describe("Rating from 0.0 to 5.0"))
  use address <- js.field("address", address_schema())
  use tags <- js.field("tags", js.array(of: tag_schema()) |> js.describe("Categorization tags"))
  use website <- js.optional("website", js.string())
  use phone <- js.optional_or_null("phone", js.string())
  js.success(Company(name:, founded_year:, public:, rating:, address:, tags:, website:, phone:))
}
```

`js.to_string(company_schema())` produces:

```json
{
  "type": "object",
  "properties": {
    "name": { "type": "string", "description": "Legal company name" },
    "founded_year": { "type": "integer", "description": "Year the company was founded" },
    "public": { "type": "boolean", "description": "Whether publicly traded" },
    "rating": { "type": ["number", "null"], "description": "Rating from 0.0 to 5.0" },
    "address": {
      "type": "object",
      "properties": {
        "street": { "type": "string", "description": "Street address" },
        "city": { "type": "string" },
        "zip": { "type": "string", "description": "ZIP or postal code" }
      },
      "required": ["street", "city"]
    },
    "tags": {
      "type": "array",
      "items": {
        "type": "object",
        "properties": {
          "key": { "type": "string" },
          "value": { "type": "string" }
        },
        "required": ["key", "value"]
      },
      "description": "Categorization tags"
    },
    "website": { "type": "string" },
    "phone": { "type": ["string", "null"] }
  },
  "required": ["name", "founded_year", "public", "address", "tags"]
}
```

The same schema decodes JSON into typed Gleam values:

```gleam
// All fields present
js.decode(company_schema(), from: "{\"name\":\"Acme Corp\",\"founded_year\":1995,\"public\":true,\"rating\":4.5,\"address\":{\"street\":\"123 Main St\",\"city\":\"Springfield\",\"zip\":\"62704\"},\"tags\":[{\"key\":\"industry\",\"value\":\"tech\"}],\"website\":\"https://acme.example.com\",\"phone\":\"+1-555-0100\"}")
// -> Ok(Company(name: "Acme Corp", founded_year: 1995, public: True, rating: Some(4.5), ...))

// Only required fields
js.decode(company_schema(), from: "{\"name\":\"Tiny LLC\",\"founded_year\":2020,\"public\":false,\"address\":{\"street\":\"1 Elm St\",\"city\":\"Shelbyville\"},\"tags\":[]}")
// -> Ok(Company(name: "Tiny LLC", ..., rating: None, website: None, phone: None))

// Explicit nulls
js.decode(company_schema(), from: "{\"name\":\"Null Inc\",\"founded_year\":2010,\"public\":true,\"rating\":null,\"address\":{\"street\":\"0 Zero Rd\",\"city\":\"Nowhere\"},\"tags\":[],\"phone\":null}")
// -> Ok(Company(name: "Null Inc", ..., rating: None, phone: None))
```

## API reference

### Primitives

| Function | Type | JSON Schema |
|---|---|---|
| `js.string()` | `JsonSchema(String)` | `{"type": "string"}` |
| `js.integer()` | `JsonSchema(Int)` | `{"type": "integer"}` |
| `js.number()` | `JsonSchema(Float)` | `{"type": "number"}` |
| `js.boolean()` | `JsonSchema(Bool)` | `{"type": "boolean"}` |

### Composites

| Function | Type | JSON Schema |
|---|---|---|
| `js.array(of: schema)` | `JsonSchema(List(t))` | `{"type": "array", "items": ...}` |
| `js.nullable(schema)` | `JsonSchema(Option(t))` | `{"type": ["<t>", "null"]}` |

### Object fields

| Function | Required? | Nullable? | Gleam type |
|---|---|---|---|
| `js.field` | yes | no | `t` |
| `js.optional` | no | no | `Option(t)` |
| `js.optional_or_null` | no | yes | `Option(t)` |
| `js.field_with_default` | no | no | `t` (uses default when absent) |

All four are used with Gleam's `use` syntax to chain fields:

```gleam
use value <- js.field("name", js.string())
use value <- js.optional("name", js.string())
use value <- js.optional_or_null("name", js.string())
use value <- js.field_with_default("port", js.integer(), default: 8080, encode: json.int)
```

### Enum / Const

| Function | Type | JSON Schema |
|---|---|---|
| `js.enum(["a", "b"])` | `JsonSchema(String)` | `{"type": "string", "enum": ["a", "b"]}` |
| `js.enum_map([#("a", A), #("b", B)])` | `JsonSchema(t)` | `{"type": "string", "enum": ["a", "b"]}` |
| `js.constant("a")` | `JsonSchema(String)` | `{"type": "string", "const": "a"}` |
| `js.constant_map("a", A)` | `JsonSchema(t)` | `{"type": "string", "const": "a"}` |

The `_map` variants decode to a custom Gleam type instead of `String`:

```gleam
type Color { Red Green Blue }

// Decodes to String
js.enum(["red", "green", "blue"])

// Decodes to Color
js.enum_map([#("red", Red), #("green", Green), #("blue", Blue)])
```

### Annotations

```gleam
js.string() |> js.describe("A human-readable description")
```

### Operations

```gleam
js.to_string(schema)                    // -> String (JSON Schema)
js.to_json(schema)                      // -> json.Json (for embedding in larger structures)
js.decode(schema, from: json_string)    // -> Result(t, json.DecodeError)
```

## JSON Schema coverage

| Feature | Status | Notes |
|---|---|---|
| **Types** | | |
| `string` | Supported | |
| `integer` | Supported | |
| `number` | Supported | |
| `boolean` | Supported | |
| `array` | Supported | |
| `object` | Supported | Nested objects, required/optional fields |
| `null` / nullable | Supported | Via `nullable`, `optional_or_null` |
| `enum` | Supported | String values via `enum`, `enum_map` |
| `const` | Supported | String values via `constant`, `constant_map` |
| **Composition** | | |
| `oneOf` | Not yet | Discriminated unions |
| `anyOf` | Not yet | Union types |
| `allOf` | Not yet | Intersection / schema merging |
| `not` | Not yet | Negation |
| `$ref` / `$defs` | Not yet | Reusable schema definitions |
| **Object keywords** | | |
| `properties` | Supported | |
| `required` | Supported | |
| `additionalProperties` | Not yet | |
| `patternProperties` | Not yet | |
| `propertyNames` | Not yet | |
| `minProperties` / `maxProperties` | Not yet | |
| `dependentRequired` / `dependentSchemas` | Not yet | |
| **Array keywords** | | |
| `items` | Supported | |
| `prefixItems` | Not yet | Tuple validation |
| `minItems` / `maxItems` | Not yet | |
| `uniqueItems` | Not yet | |
| `contains` | Not yet | |
| **String validation** | | |
| `minLength` / `maxLength` | Not yet | |
| `pattern` | Not yet | Regex |
| `format` | Not yet | email, uri, date-time, etc. |
| **Number validation** | | |
| `minimum` / `maximum` | Not yet | |
| `exclusiveMinimum` / `exclusiveMaximum` | Not yet | |
| `multipleOf` | Not yet | |
| **Annotations** | | |
| `description` | Supported | Via `describe` |
| `title` | Not yet | |
| `default` | Supported | Via `field_with_default` |
| `examples` | Not yet | |
| `deprecated` | Not yet | |
| `readOnly` / `writeOnly` | Not yet | |
| **Conditional** | | |
| `if` / `then` / `else` | Not yet | |
| **Meta** | | |
| `$schema` | Not yet | Draft identifier |
| `$id` | Not yet | |
| `$comment` | Not yet | |

## Compatibility

- Requires `gleam_json` >= 3.0
- Works on both Erlang and JavaScript targets
