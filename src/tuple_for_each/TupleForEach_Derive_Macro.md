# TupleForEach Derive Macro

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs)

This document provides comprehensive reference documentation for the `TupleForEach` derive macro, which is the core procedural macro that generates iteration utilities for tuple structs. For information about the generated macros themselves, see [Generated Macros](/arceos-org/tuple_for_each/5.2-generated-macros). For information about the generated methods, see [Generated Methods](/arceos-org/tuple_for_each/5.3-generated-methods).

## Purpose and Scope

The `TupleForEach` derive macro is a procedural macro that automatically generates iteration utilities for tuple structs at compile time. When applied to a tuple struct, it creates field iteration macros and utility methods that enable ergonomic access to tuple fields without manual indexing.

## Macro Declaration

The derive macro is declared as a procedural macro using the `#[proc_macro_derive]` attribute:

```rust
#[proc_macro_derive(TupleForEach)]
pub fn tuple_for_each(item: TokenStream) -> TokenStream
```

**Entry Point Flow**

```mermaid
flowchart TD
Input["TokenStream input"]
Parse["syn::parse_macro_input!(item as DeriveInput)"]
Check["ast.data matches Data::Struct?"]
Error1["Compile Error"]
Fields["strct.fields matches Fields::Unnamed?"]
Error2["Compile Error: 'can only be attached to tuple structs'"]
Generate["impl_for_each(&ast, strct)"]
Output["Generated TokenStream"]
CompileError["Error::new_spanned().to_compile_error()"]

Check --> Error1
Check --> Fields
Error1 --> CompileError
Error2 --> CompileError
Fields --> Error2
Fields --> Generate
Generate --> Output
Input --> Parse
Parse --> Check
```

Sources: [src/lib.rs(L11 - L24)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L11-L24)

## Requirements and Constraints

### Valid Input Types

The derive macro can only be applied to **tuple structs** - structs with unnamed fields. The validation logic enforces two conditions:

|Condition|Requirement|Error if Failed|
| --- | --- | --- |
|Struct Type|Must beData::Structvariant|Compile error with span|
|Field Type|Must haveFields::Unnamed|"attributetuple_for_eachcan only be attached to tuple structs"|

### Supported Tuple Struct Examples

```css
// ✅ Valid - basic tuple struct
#[derive(TupleForEach)]
struct Point(i32, i32);

// ✅ Valid - mixed types
#[derive(TupleForEach)]
struct Mixed(String, i32, bool);

// ✅ Valid - generic tuple struct
#[derive(TupleForEach)]
struct Generic<T, U>(T, U);

// ❌ Invalid - named struct
#[derive(TupleForEach)]
struct Named { x: i32, y: i32 }

// ❌ Invalid - unit struct
#[derive(TupleForEach)]
struct Unit;
```

**Validation Logic Flow**

```mermaid
flowchart TD
ast["DeriveInput AST"]
data_check["ast.data == Data::Struct?"]
error_span["Error::new_spanned(ast, message)"]
strct["Extract DataStruct"]
fields_check["strct.fields == Fields::Unnamed?"]
error_msg["'can only be attached to tuple structs'"]
impl_call["impl_for_each(&ast, strct)"]
compile_error["to_compile_error().into()"]
success["Generated Code TokenStream"]

ast --> data_check
data_check --> error_span
data_check --> strct
error_msg --> compile_error
error_span --> compile_error
fields_check --> error_msg
fields_check --> impl_call
impl_call --> success
strct --> fields_check
```

Sources: [src/lib.rs(L13 - L23)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L13-L23)

## Code Generation Process

### Name Resolution

The macro converts tuple struct names from PascalCase to snake_case using the `pascal_to_snake` function to create macro names:

|Tuple Struct Name|Generated Macro Prefix|For Each Macro|Enumerate Macro|
| --- | --- | --- | --- |
|Point|point|point_for_each!|point_enumerate!|
|MyTuple|my_tuple|my_tuple_for_each!|my_tuple_enumerate!|
|HttpResponse|http_response|http_response_for_each!|http_response_enumerate!|

### Generated Code Structure

For each tuple struct, the macro generates:

1. **Utility Methods**: `len()` and `is_empty()` implementations
2. **For Each Macro**: Field iteration with immutable and mutable variants
3. **Enumerate Macro**: Indexed field iteration with immutable and mutable variants

**Code Generation Pipeline**

```mermaid
flowchart TD
ast_input["DeriveInput & DataStruct"]
name_conv["pascal_to_snake(tuple_name)"]
macro_names["format_ident! for macro names"]
field_count["strct.fields.len()"]
iter_gen["Generate field iteration code"]
for_each_imm["for_each: &$tuple.#idx"]
for_each_mut["for_each_mut: &mut $tuple.#idx"]
enum_imm["enumerate: (#idx, &$tuple.#idx)"]
enum_mut["enumerate_mut: (#idx, &mut $tuple.#idx)"]
macro_def["macro_rules! definitions"]
methods["impl block with len() & is_empty()"]
quote_expand["quote! macro expansion"]
token_stream["Final TokenStream"]

ast_input --> name_conv
enum_imm --> macro_def
enum_mut --> macro_def
field_count --> iter_gen
field_count --> methods
for_each_imm --> macro_def
for_each_mut --> macro_def
iter_gen --> enum_imm
iter_gen --> enum_mut
iter_gen --> for_each_imm
iter_gen --> for_each_mut
macro_def --> quote_expand
macro_names --> field_count
methods --> quote_expand
name_conv --> macro_names
quote_expand --> token_stream
```

Sources: [src/lib.rs(L58 - L122)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L58-L122)

### Field Access Pattern Generation

The macro generates field access code for each tuple field using zero-based indexing:

```
// For a 3-field tuple struct, generates:
// Field 0: &$tuple.0, &mut $tuple.0
// Field 1: &$tuple.1, &mut $tuple.1  
// Field 2: &$tuple.2, &mut $tuple.2
```

The field iteration logic creates four variants of access patterns:

|Pattern Type|Mutability|Generated Code Template|
| --- | --- | --- |
|for_each|Immutable|{ let $item = &$tuple.#idx; $code }|
|for_each_mut|Mutable|{ let $item = &mut $tuple.#idx; $code }|
|enumerate|Immutable|{ let $idx = #idx; let $item = &$tuple.#idx; $code }|
|enumerate_mut|Mutable|{ let $idx = #idx; let $item = &mut $tuple.#idx; $code }|

Sources: [src/lib.rs(L64 - L83)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L64-L83)

## Error Handling

The derive macro implements compile-time validation with descriptive error messages:

### Error Cases

1. **Non-Struct Types**: Applied to enums, unions, or other non-struct types
2. **Named Structs**: Applied to structs with named fields instead of tuple structs
3. **Unit Structs**: Applied to unit structs (no fields)

### Error Message Generation

```yaml
Error::new_spanned(
    ast,
    "attribute `tuple_for_each` can only be attached to tuple structs",
)
.to_compile_error()
.into()
```

The error uses `syn::Error::new_spanned` to provide precise source location information and converts to a `TokenStream` containing the compile error.

**Error Handling Flow**

```mermaid
flowchart TD
invalid_input["Invalid Input Structure"]
new_spanned["Error::new_spanned(ast, message)"]
error_message["'attribute tuple_for_each can only be attached to tuple structs'"]
to_compile_error["to_compile_error()"]
into_token_stream["into()"]
compiler_error["Compiler displays error at source location"]

error_message --> to_compile_error
into_token_stream --> compiler_error
invalid_input --> new_spanned
new_spanned --> error_message
to_compile_error --> into_token_stream
```

Sources: [src/lib.rs(L18 - L23)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L18-L23)

## Implementation Dependencies

The derive macro relies on several key crates for parsing and code generation:

|Dependency|Purpose|Usage in Code|
| --- | --- | --- |
|syn|AST parsing|syn::parse_macro_input!,DeriveInput,DataStruct|
|quote|Code generation|quote!macro,format_ident!|
|proc_macro2|Token stream handling|Return type forimpl_for_each|
|proc_macro|Procedural macro interface|TokenStreaminput/output|

Sources: [src/lib.rs(L3 - L5)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L3-L5)