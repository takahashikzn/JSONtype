# RFC: JSONtype — JSON with Native Type Support

## Abstract
JSONtype is a data interchange format that extends JSON with a single syntactic addition: `Type(value)`. This notation introduces typed values while preserving JSON's simplicity. JSONtype is a strict superset of JSON: every valid JSON document is also a valid JSONtype document. Unknown types are always unwrapped to their inner value, ensuring graceful degradation.

## 1. Introduction

### 1.1 Motivation
JSON has become the de facto standard for data interchange, but it lacks native support for common data types such as dates, big integers, and decimals. Existing workarounds involve string encodings, schema annotations, or verbose wrappers, which are often ambiguous and inconvenient.

JSONtype addresses this limitation with a single minimal extension.
Prior attempts often failed by aiming for a “perfect typing system”.
JSONtype aims to take the opposite approach: 80% of the benefit with 1% of the complexity.

JSONtype respects JSON’s lightweight nature: it is not a full type system, but a natural way to annotate values, conveying intent without adding noise.

### 1.2 Design Goals
1. **Minimalism**: Only one new syntax form (`Type(value)`)
2. **Superset**: All valid JSON is valid JSONtype
3. **Human-readable**: Intuitive and clear representation
4. **Extensible**: Arbitrary user-defined types
5. **Graceful degradation**: Unknown types unwrap to their value

## 2. Specification

### 2.1 Syntax
JSONtype extends the JSON grammar with a `typed-value`. Each `typed-value` wraps exactly one `value`.

(ABNF; RFC 5234)
```
JSONTYPE = VALUE

VALUE    = OBJECT / ARRAY / STRING / NUMBER / BOOLEAN / NULL / TYPED
TYPED    = TYPENAME "(" VALUE ")"

TYPENAME = UPPER *( ALPHA / DIGIT / "." )
UPPER    = %x41-5A            ; A-Z
ALPHA    = %x41-5A / %x61-7A  ; A-Z / a-z
DIGIT    = %x30-39
```

### 2.2 Core Types
Parsers SHOULD recognize the following core types:

- **Date**: RFC 3339 / ISO 8601 strings.  
  Examples: `Date("2025-09-01")`, `Date("2025-09-01T12:00:00Z")`
- **BigInt**: Arbitrary-precision integers encoded as strings.  
  Example: `BigInt("9007199254740993")`
- **Decimal**: Arbitrary-precision decimals encoded as strings, preserving scale.  
  Example: `Decimal("99.99")`, `Decimal("1.230")`

### 2.3 User-Defined Types
Any other type name is treated as user-defined. The value can be any JSON/JSONtype value. For multi-field data, use an object:

```js
User({"id": 123, "name": "kaz"})
Point({"x": 35.6762, "y": 139.6503})
Money({"amount": Decimal("1000.00"), "currency": "JPY"})
```

### 2.4 Namespaced Types
Dot notation MAY be used for namespacing:

```js
blog.User({"id": 1})
shop.User({"id": 2})
```

## 3. Parsing Behavior

### 3.1 Type-Aware Parsers
Parsers MUST:
1. Recognize `Type(value)`
2. Interpret known core types into native representations
3. Provide extension hooks for user-defined types

### 3.2 Unknown Types
Parsers MUST unwrap unknown types to their inner value:

```js
UnknownType({"foo": 1, "bar": 2}) -> {"foo": 1, "bar": 2}
Wrapper(List([1,2,3])) -> [1,2,3]
```

This design allows type names to serve as lightweight annotations: even if a type is not defined, the surrounding context can still benefit from the label while the actual data remains usable. In other words, type names may also act as comments without breaking compatibility.

### 3.3 Compatibility
- All valid JSON is valid JSONtype.
- JSONtype documents containing `Type(value)` are NOT valid plain JSON, by design.

## 4. Examples

```js
{
  "id": BigInt("9007199254740993"),
  "created": Date("2025-09-01T12:00:00Z"),
  "price": Decimal("99.99"),
  "user": User({
    "name": "kaz",
    "birthday": Date("1990-01-01")
  }),
  "location": Point({"x": 35.6762, "y": 139.6503})
}
```

## 5. Implementation Considerations
- **File extension**: `.jsont`  
- **Media type**: `application/jsontype`  
- **Error handling**: Invalid core type formats MUST produce errors. Unknown types are always unwrapped.

## 6. Security Considerations
- Type handlers MUST NOT execute arbitrary code.  
- Parsers SHOULD enforce limits on nesting depth, token count, and string length.  
- Date/Decimal/BigInt MUST be validated strictly.
- Implementations SHOULD provide a configuration option to reject user-defined type handlers for untrusted input, treating all unknown types as simple unwrap. This prevents potential deserialization vulnerabilities.

## 7. Comparison (Informative)

| Feature          | JSON | JSON5 | ExtJSON | JSONtype |
|------------------|------|-------|---------|----------|
| Date             | No   | No    | Yes     | Yes      |
| BigInt           | No   | No    | Yes     | Yes      |
| Decimal          | No   | No    | Yes     | Yes      |
| User Types       | No   | No    | No      | Yes      |
| Superset of JSON | Yes  | Yes   | No      | Yes      |
| Unknown Types    | —    | —     | —       | Unwrap   |

## 8. References
- RFC 8259: The JavaScript Object Notation (JSON)  
- ECMA-404: The JSON Data Interchange Format  
- ISO 8601, RFC 3339: Date and Time on the Internet  

---
Author: takahashikzn  
Co-Author: ChatGPT-5  
Status: Draft  
Version: 1.0  
Date: 2025-09-01  
License: MIT  
