# Type System

> **Relevant source files**
> * [axconfig-gen/src/tests.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/tests.rs)
> * [axconfig-gen/src/ty.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs)
> * [axconfig-gen/src/value.rs](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs)

## Purpose and Scope

The Type System is the core component responsible for type inference, validation, and code generation in the axconfig-gen library. It provides mechanisms to parse type annotations from strings, infer types from TOML values, validate type compatibility, and generate appropriate Rust type declarations for configuration values.

For information about how types are used in configuration data structures, see [Core Data Structures](/arceos-org/axconfig-gen/2.2.1-core-data-structures). For details on how the type system integrates with output generation, see [Output Generation](/arceos-org/axconfig-gen/2.2.3-output-generation).

## ConfigType Enumeration

The foundation of the type system is the `ConfigType` enum, which represents all supported configuration value types in the system.

```mermaid
flowchart TD
ConfigType["ConfigType enum"]
Bool["Bool: boolean values"]
Int["Int: signed integers"]
Uint["Uint: unsigned integers"]
String["String: string values"]
Tuple["Tuple(Vec<ConfigType>): fixed-length heterogeneous sequences"]
Array["Array(Box<ConfigType>): variable-length homogeneous sequences"]
Unknown["Unknown: type to be inferred"]
TupleExample["Example: (int, str, bool)"]
ArrayExample["Example: [uint] or [[str]]"]

Array --> ArrayExample
ConfigType --> Array
ConfigType --> Bool
ConfigType --> Int
ConfigType --> String
ConfigType --> Tuple
ConfigType --> Uint
ConfigType --> Unknown
Tuple --> TupleExample
```

**Supported Type Categories**

|Type Category|ConfigType Variant|Rust Equivalent|TOML Representation|
| --- | --- | --- | --- |
|Boolean|Bool|bool|true,false|
|Signed Integer|Int|isize|-123,0x80|
|Unsigned Integer|Uint|usize|123,0xff|
|String|String|&str|"hello"|
|Tuple|Tuple(Vec<ConfigType>)|(T1, T2, ...)|[val1, val2, ...]|
|Array|Array(Box<ConfigType>)|&[T]|[val1, val2, ...]|
|Unknown|Unknown|N/A|Used for inference|

Sources: [axconfig-gen/src/ty.rs(L4 - L22)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs#L4-L22)

## Type Parsing and String Representation

The type system supports parsing type specifications from string format, enabling type annotations in TOML comments and explicit type declarations.

```mermaid
flowchart TD
TypeString["Type String Input"]
Parser["ConfigType::new()"]
BasicTypes["Basic Typesbool, int, uint, str"]
TupleParser["Tuple Parser(type1, type2, ...)"]
ArrayParser["Array Parser[element_type]"]
TupleSplitter["split_tuple_items()Handle nested parentheses"]
ElementType["Recursive parsingfor element type"]
ConfigTypeResult["ConfigType instance"]
DisplayFormat["Display traitHuman-readable output"]
RustType["to_rust_type()Rust code generation"]

ArrayParser --> ElementType
BasicTypes --> ConfigTypeResult
ConfigTypeResult --> DisplayFormat
ConfigTypeResult --> RustType
ElementType --> ConfigTypeResult
Parser --> ArrayParser
Parser --> BasicTypes
Parser --> TupleParser
TupleParser --> TupleSplitter
TupleSplitter --> ConfigTypeResult
TypeString --> Parser
```

**Type String Examples**

|Input String|Parsed Type|Rust Output|
| --- | --- | --- |
|"bool"|ConfigType::Bool|"bool"|
|"[uint]"|ConfigType::Array(Box::new(ConfigType::Uint))|"&[usize]"|
|"(int, str)"|ConfigType::Tuple(vec![ConfigType::Int, ConfigType::String])|"(isize, &str)"|
|"[[bool]]"|ConfigType::Array(Box::new(ConfigType::Array(...)))|"&[&[bool]]"|

Sources: [axconfig-gen/src/ty.rs(L24 - L61)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs#L24-L61) [axconfig-gen/src/ty.rs(L83 - L111)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs#L83-L111) [axconfig-gen/src/ty.rs(L113 - L134)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs#L113-L134)

## Type Inference System

The type inference system automatically determines types from TOML values, supporting both simple and complex nested structures.

```mermaid
flowchart TD
TOMLValue["TOML Value"]
BooleanValue["Boolean Value"]
IntegerValue["Integer Value"]
StringValue["String Value"]
ArrayValue["Array Value"]
BoolType["ConfigType::Bool"]
IntegerCheck["Check sign"]
NegativeInt["< 0: ConfigType::Int"]
PositiveInt["Unsupported markdown: blockquote"]
NumericString["is_num() check"]
StringAsUint["Numeric: ConfigType::Uint"]
StringAsStr["Non-numeric: ConfigType::String"]
InferElements["Infer element types"]
AllSame["All elements same type?"]
ArrayResult["Yes: ConfigType::Array"]
TupleResult["No: ConfigType::Tuple"]
UnknownResult["Empty/Unknown: ConfigType::Unknown"]

AllSame --> ArrayResult
AllSame --> TupleResult
AllSame --> UnknownResult
ArrayValue --> InferElements
BooleanValue --> BoolType
InferElements --> AllSame
IntegerCheck --> NegativeInt
IntegerCheck --> PositiveInt
IntegerValue --> IntegerCheck
NumericString --> StringAsStr
NumericString --> StringAsUint
StringValue --> NumericString
TOMLValue --> ArrayValue
TOMLValue --> BooleanValue
TOMLValue --> IntegerValue
TOMLValue --> StringValue
```

**Inference Rules**

* **Integers**: Negative values infer as `Int`, non-negative as `Uint`
* **Strings**: Numeric strings (hex, binary, octal, decimal) infer as `Uint`, others as `String`
* **Arrays**: Homogeneous arrays become `Array`, heterogeneous become `Tuple`
* **Nested Arrays**: Recursively infer element types

Sources: [axconfig-gen/src/value.rs(L177 - L224)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L177-L224) [axconfig-gen/src/value.rs(L114 - L125)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L114-L125)

## Type Validation and Matching

The type system provides comprehensive validation to ensure values conform to expected types, with special handling for string-encoded numbers and flexible integer types.

```mermaid
flowchart TD
ValueTypeMatch["value_type_matches()"]
BoolMatch["Boolean vs Bool"]
IntMatch["Integer vs Int/Uint"]
StringMatch["String value analysis"]
ArrayMatch["Array structure validation"]
StringNumeric["is_num() check"]
StringAsNumber["Numeric stringmatches Int/Uint/String"]
StringLiteral["Non-numeric stringmatches String only"]
TupleValidation["Array vs TupleLength and element validation"]
ArrayValidation["Array vs ArrayElement type validation"]
Elementwise["Check each elementagainst corresponding type"]
Homogeneous["Check all elementsagainst element type"]

ArrayMatch --> ArrayValidation
ArrayMatch --> TupleValidation
ArrayValidation --> Homogeneous
StringMatch --> StringNumeric
StringNumeric --> StringAsNumber
StringNumeric --> StringLiteral
TupleValidation --> Elementwise
ValueTypeMatch --> ArrayMatch
ValueTypeMatch --> BoolMatch
ValueTypeMatch --> IntMatch
ValueTypeMatch --> StringMatch
```

**Type Compatibility Matrix**

|TOML Value|Bool|Int|Uint|String|Notes|
| --- | --- | --- | --- | --- | --- |
|true/false|✓|✗|✗|✗|Exact match only|
|123|✗|✓|✓|✗|Integers match both Int/Uint|
|-123|✗|✓|✓|✗|Negative still matches Uint|
|"123"|✗|✓|✓|✓|Numeric strings are flexible|
|"abc"|✗|✗|✗|✓|Non-numeric strings|

Sources: [axconfig-gen/src/value.rs(L142 - L175)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L142-L175) [axconfig-gen/src/value.rs(L58 - L80)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L58-L80)

## Integration with ConfigValue

The type system integrates closely with `ConfigValue` to provide type-safe configuration value management.

```mermaid
flowchart TD
ConfigValue["ConfigValue struct"]
Value["value: toml_edit::Value"]
OptionalType["ty: Option<ConfigType>"]
ExplicitType["Explicit typefrom constructor"]
InferredType["Inferred typefrom value analysis"]
NoType["No typepure value storage"]
TypeMethods["Type-related methods"]
InferredTypeMethod["inferred_type()Get computed type"]
TypeMatchesMethod["type_matches()Validate against type"]
UpdateMethod["update()Type-safe value updates"]
ToRustMethod["to_rust_value()Generate Rust code"]

ConfigValue --> OptionalType
ConfigValue --> TypeMethods
ConfigValue --> Value
OptionalType --> ExplicitType
OptionalType --> InferredType
OptionalType --> NoType
TypeMethods --> InferredTypeMethod
TypeMethods --> ToRustMethod
TypeMethods --> TypeMatchesMethod
TypeMethods --> UpdateMethod
```

**ConfigValue Type Operations**

* **Construction**: Values can be created with or without explicit types
* **Inference**: Types can be computed from TOML values automatically
* **Validation**: Updates are validated against existing or specified types
* **Code Generation**: Type information drives Rust code output format

Sources: [axconfig-gen/src/value.rs(L7 - L12)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L7-L12) [axconfig-gen/src/value.rs(L52 - L103)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L52-L103)

## Rust Code Generation

The type system drives generation of appropriate Rust type declarations and value literals for compile-time configuration constants.

```mermaid
flowchart TD
TypeToRust["Type → Rust Mapping"]
BasicMapping["Basic Type Mapping"]
ComplexMapping["Complex Type Mapping"]
BoolToRust["Bool → bool"]
IntToRust["Int → isize"]
UintToRust["Uint → usize"]
StringToRust["String → &str"]
TupleToRust["Tuple → (T1, T2, ...)"]
ArrayToRust["Array → &[T]"]
TupleElements["Recursive element mapping"]
ArrayElement["Element type mapping"]
ValueToRust["Value → Rust Code"]
LiteralMapping["Literal Value Mapping"]
StructureMapping["Structure Mapping"]
BoolLiteral["true/false"]
NumericLiteral["Integer literals"]
StringLiteral["String literals"]
TupleStructure["(val1, val2, ...)"]
ArrayStructure["&[val1, val2, ...]"]

ArrayToRust --> ArrayElement
BasicMapping --> BoolToRust
BasicMapping --> IntToRust
BasicMapping --> StringToRust
BasicMapping --> UintToRust
ComplexMapping --> ArrayToRust
ComplexMapping --> TupleToRust
LiteralMapping --> BoolLiteral
LiteralMapping --> NumericLiteral
LiteralMapping --> StringLiteral
StructureMapping --> ArrayStructure
StructureMapping --> TupleStructure
TupleToRust --> TupleElements
TypeToRust --> BasicMapping
TypeToRust --> ComplexMapping
ValueToRust --> LiteralMapping
ValueToRust --> StructureMapping
```

**Rust Generation Examples**

|ConfigType|Rust Type|Example Value|Generated Rust|
| --- | --- | --- | --- |
|Bool|bool|true|true|
|Uint|usize|"0xff"|0xff|
|Array(String)|&[&str]|["a", "b"]|&["a", "b"]|
|Tuple([Uint, String])|(usize, &str)|[123, "test"]|(123, "test")|

Sources: [axconfig-gen/src/ty.rs(L62 - L81)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/ty.rs#L62-L81) [axconfig-gen/src/value.rs(L243 - L288)&emsp;](https://github.com/arceos-org/axconfig-gen/blob/99357274/axconfig-gen/src/value.rs#L243-L288)