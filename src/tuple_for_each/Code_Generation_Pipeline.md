# Code Generation Pipeline

> **Relevant source files**
> * [src/lib.rs](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs)

This document details the core code generation logic within the `tuple_for_each` crate, specifically focusing on the `impl_for_each` function and its supporting components. This covers the transformation process from parsed AST input to generated Rust code output, including field iteration patterns, macro template creation, and documentation generation.

For information about the initial parsing and validation phase, see [Derive Macro Processing](/arceos-org/tuple_for_each/3.1-derive-macro-processing). For details about the generated API surface, see [Generated Functionality](/arceos-org/tuple_for_each/2.2-generated-functionality).

## Pipeline Overview

The code generation pipeline transforms a validated tuple struct AST into executable Rust code through a series of well-defined steps. The process centers around the `impl_for_each` function, which orchestrates the entire generation workflow.

### Code Generation Flow

```mermaid
flowchart TD
A["tuple_for_each()"]
B["impl_for_each()"]
C["pascal_to_snake()"]
D["Field Access Generation"]
E["Macro Template Creation"]
F["Documentation Generation"]
G["macro_name: String"]
H["for_each: Vec"]
I["for_each_mut: Vec"]
J["enumerate: Vec"]
K["enumerate_mut: Vec"]
L["macro_for_each!"]
M["macro_enumerate!"]
N["len() method"]
O["is_empty() method"]
P["macro_for_each_doc"]
Q["macro_enumerate_doc"]
R["Final TokenStream"]

A --> B
B --> C
B --> D
B --> E
B --> F
C --> G
D --> H
D --> I
D --> J
D --> K
E --> L
E --> M
E --> N
E --> O
F --> P
F --> Q
G --> L
G --> M
H --> L
I --> L
J --> M
K --> M
L --> R
M --> R
N --> R
O --> R
P --> L
Q --> M
```

Sources: [src/lib.rs(L58 - L122)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L58-L122)

## Name Conversion Process

The pipeline begins by converting the tuple struct's PascalCase name to snake_case for macro naming. This conversion is handled by the `pascal_to_snake` utility function.

### Conversion Algorithm

|Input (PascalCase)|Output (snake_case)|Generated Macro Names|
| --- | --- | --- |
|MyTuple|my_tuple|my_tuple_for_each!,my_tuple_enumerate!|
|HTTPResponse|h_t_t_p_response|h_t_t_p_response_for_each!,h_t_t_p_response_enumerate!|
|SimpleStruct|simple_struct|simple_struct_for_each!,simple_struct_enumerate!|

The conversion process inserts underscores before uppercase characters (except the first character) and converts all characters to lowercase:

```mermaid
flowchart TD
A["tuple_name.to_string()"]
B["pascal_to_snake()"]
C["macro_name: String"]
D["format_ident!('{}_for_each', macro_name)"]
E["format_ident!('{}_enumerate', macro_name)"]
F["macro_for_each: Ident"]
G["macro_enumerate: Ident"]

A --> B
B --> C
C --> D
C --> E
D --> F
E --> G
```

Sources: [src/lib.rs(L60 - L62)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L60-L62) [src/lib.rs(L124 - L133)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L124-L133)

## Field Access Code Generation

The core of the pipeline generates field access patterns for each tuple field. This process creates four distinct code patterns to handle different iteration scenarios.

### Field Iteration Logic

```mermaid
flowchart TD
A["field_num = strct.fields.len()"]
B["for i in 0..field_num"]
C["idx = Index::from(i)"]
D["for_each.push()"]
E["for_each_mut.push()"]
F["enumerate.push()"]
G["enumerate_mut.push()"]
H["&$tuple.#idx"]
I["&mut $tuple.#idx"]
J["($idx = #idx, &$tuple.#idx)"]
K["($idx = #idx, &mut $tuple.#idx)"]

A --> B
B --> C
C --> D
C --> E
C --> F
C --> G
D --> H
E --> I
F --> J
G --> K
```

Each field access pattern is generated using the `quote!` macro to create `TokenStream` fragments:

|Pattern Type|Code Template|Variable Binding|
| --- | --- | --- |
|for_each|let $item = &$tuple.#idx; $code|Immutable reference|
|for_each_mut|let $item = &mut $tuple.#idx; $code|Mutable reference|
|enumerate|let $idx = #idx; let $item = &$tuple.#idx; $code|Index + immutable reference|
|enumerate_mut|let $idx = #idx; let $item = &mut $tuple.#idx; $code|Index + mutable reference|

Sources: [src/lib.rs(L64 - L83)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L64-L83)

## Macro Template Creation

The generated field access patterns are assembled into complete macro definitions using Rust's `macro_rules!` system. Each macro supports both immutable and mutable variants through pattern matching.

### Macro Structure Assembly

```mermaid
flowchart TD
A["for_each patterns"]
B["macro_for_each definition"]
C["enumerate patterns"]
D["macro_enumerate definition"]
E["($item:ident in $tuple:ident $code:block)"]
F["($item:ident in mut $tuple:ident $code:block)"]
G["(($idx:ident, $item:ident) in $tuple:ident $code:block)"]
H["(($idx:ident, $item:ident) in mut $tuple:ident $code:block)"]
I["#(#for_each)*"]
J["#(#for_each_mut)*"]
K["#(#enumerate)*"]
L["#(#enumerate_mut)*"]

A --> B
B --> E
B --> F
C --> D
D --> G
D --> H
E --> I
F --> J
G --> K
H --> L
```

The macro definitions use repetition syntax (`#()*`) to expand the vector of field access patterns into sequential code blocks. This creates the illusion of iteration over heterogeneous tuple fields at compile time.

Sources: [src/lib.rs(L102 - L120)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L102-L120)

## Documentation Generation

The pipeline includes automated documentation generation for the created macros. The `gen_doc` function creates formatted documentation strings with usage examples.

### Documentation Template System

|Documentation Type|Template Function|Generated Content|
| --- | --- | --- |
|for_each|gen_doc("for_each", tuple_name, macro_name)|Usage examples with field iteration|
|enumerate|gen_doc("enumerate", tuple_name, macro_name)|Usage examples with index + field iteration|

The documentation templates include:

* Description of macro purpose
* Reference to the derive macro that generated it
* Code examples showing proper usage syntax
* Placeholder substitution for tuple and macro names

```mermaid
flowchart TD
A["tuple_name.to_string()"]
B["gen_doc()"]
C["macro_name"]
D["macro_for_each_doc"]
E["macro_enumerate_doc"]
F["#[doc = #macro_for_each_doc]"]
G["#[doc = #macro_enumerate_doc]"]

A --> B
B --> D
B --> E
C --> B
D --> F
E --> G
```

Sources: [src/lib.rs(L26 - L56)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L26-L56) [src/lib.rs(L85 - L86)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L85-L86)

## Final Assembly

The pipeline concludes by assembling all generated components into a single `TokenStream` using the `quote!` macro. This creates the complete implementation that will be inserted into the user's code.

### Component Integration

```mermaid
flowchart TD
subgraph subGraph2["Supporting Elements"]
    E["Documentation strings"]
    F["#[macro_export] attributes"]
end
subgraph subGraph1["Generated Macros"]
    C["macro_for_each!"]
    D["macro_enumerate!"]
end
subgraph subGraph0["Generated Methods"]
    A["len() method"]
    B["is_empty() method"]
end
G["impl #tuple_name block"]
H["Final TokenStream"]

A --> G
B --> G
C --> H
D --> H
E --> H
F --> H
G --> H
```

The final assembly includes:

* An `impl` block for the original tuple struct containing `len()` and `is_empty()` methods
* Two exported macros with complete documentation
* All necessary attributes for proper macro visibility

The entire generated code block is wrapped in a single `quote!` invocation that produces the final `proc_macro2::TokenStream` returned to the Rust compiler.

Sources: [src/lib.rs(L87 - L122)&emsp;](https://github.com/arceos-org/tuple_for_each/blob/19a3b4d3/src/lib.rs#L87-L122)