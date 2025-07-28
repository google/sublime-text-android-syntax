# A Technical Guide to Sublime Text Syntax Definitions

This document provides a comprehensive technical manual for authoring robust `.sublime-syntax` definitions, their associated `.tmPreferences` and `.sublime-settings` configuration files, and corresponding syntax test files. The content is structured for a technical audience, focusing on the formal grammar of each file type, the mechanics of the syntax highlighting state machine, and a complete reference of scope naming conventions.

## I. The `.sublime-syntax` File Format

The `.sublime-syntax` file is a YAML 1.2 document that serves as the primary mechanism for syntax highlighting in Sublime Text.[1, 2, 2] It comprises a header section for metadata and configuration, a variables section for reusable regular expression fragments, and a contexts section that defines the core parsing logic.

### 1.1. Header Specification

The header consists of top-level keys that configure the syntax definition's behavior and metadata.[1]

  * **`name`**: A string that defines the human-readable name of the syntax, displayed in the editor's user interface.[1]
  * **`scope`**: The top-level scope applied to the entire file, which serves as the root for all other scopes. The convention is `source.<lang>` for programming languages and `text.<lang>` for markup and other formats.[1, 3]
  * **`file_extensions`**: A list of file extensions (e.g., `c`, `h`) that automatically activate this syntax definition. This is the primary mechanism for syntax detection.[1, 2, 2, 2]
  * **`hidden_file_extensions`**: A list of file extensions that trigger the syntax but are not displayed in file dialogs.[1]
  * **`first_line_match`**: An Oniguruma regular expression used to inspect the first line of a file. This provides a content-based mechanism for syntax detection, which is crucial for files without extensions or those with ambiguous extensions, such as shell scripts with shebangs or files with editorconfig modelines.[1, 2, 2, 2] This creates a robust, two-tiered system for automatic syntax detection, where file extensions are the primary heuristic and content-based analysis is a necessary fallback.
  * **`variables`**: A key-value map where keys are variable names and values are regular expression fragments. These variables can be interpolated into `match` patterns using `{{varname}}` syntax, promoting reusability and significantly improving the readability of complex regexes.[1, 2, 2, 2]
  * **`hidden`**: A boolean value. If `true`, the syntax definition is not listed in the syntax menu, which is useful for syntaxes intended only for embedding within other definitions or for internal use.[1, 2]
  * **`extends`**: A string or list of strings specifying the package paths of parent `.sublime-syntax` files. This powerful feature allows a syntax to inherit all `variables` and `contexts` from a base language, enabling the creation of language supersets (e.g., TypeScript extending JavaScript) without code duplication.[1, 2]
  * **`version`**: An integer, which must be `2` for all new syntax definitions. Version 2 of the syntax engine corrects numerous inconsistencies and bugs present in the legacy version 1 engine, ensuring more predictable and accurate scope application.[1, 2]

### 1.2. YAML String Quoting and Escaping

Because `.sublime-syntax` files are YAML, a proper understanding of YAML's string handling is essential, particularly for writing regular expressions in `match` patterns. YAML offers several ways to denote strings, each with different rules for escaping special characters. Note that literal tab characters are illegal for indentation and should never be used for that purpose. [13]

  * **Plain (Unquoted) Strings**: Strings can often be written without any quotes. However, this is only safe for simple values. A string must be quoted if it contains special YAML characters like `:`, `{`, `[`, `]`, `,`, `&`, `*`, `#`, `?`, `|`, `-`, `>`, or if it could be misinterpreted as another type, such as a number (`1.0`) or a boolean (`true`, `no`). [13, 14]

  * **Single-Quoted Strings (`'...'`)**: In single-quoted strings, all characters are treated literally except for the single quote itself. To include a literal single quote, it must be doubled (`''`). [14, 15] Backslashes are treated as literal characters, meaning escape sequences like `\n` or `\d` are not interpreted. [16] This makes single quotes ideal for regular expressions, as it avoids the need to double-escape backslashes (e.g., `'\d'` instead of `"\\d"`). [17]

  * **Double-Quoted Strings (`"..."`)**: This is the most flexible quoting style. Double-quoted strings support C-style escape sequences. [14, 15] A backslash (`\`) is used to escape special characters, such as `\"` for a double quote, `\\` for a backslash, and `\n` for a newline. [14, 18] This style is necessary if the regular expression itself must contain a single quote.

  * **Block Scalars (`|` and `>`)**: For multi-line strings, YAML provides block scalars. These are commonly used for the `first_line_match` key.

      * **Literal Style (`|`)**: Preserves all newlines within the block as literal newline characters (`\n`). [19, 20]
      * **Folded Style (`>`)**: Folds newlines within the block into spaces, which is useful for breaking up long lines for readability without affecting the string's value. [19, 20]
      * **Chomping Indicators**: These styles can be followed by a chomping indicator (`-` or `+`) to control the handling of trailing newlines at the end of the block. [20, 21]

**Recommendation**: For `match` patterns, prefer single quotes to avoid double-escaping backslashes. If the regex contains a single quote, use double quotes. If a regex contains any special YAML characters (e.g., `#`, `:`, `-`, `{`, `[`), it must be quoted. [22]

### 1.3. The Oniguruma Regular Expression Engine

Sublime Text's syntax highlighting engine is powered by the Oniguruma regular expression library, the same engine historically used by the Ruby programming language.[4, 5, 6] Proficiency in its specific features is a prerequisite for authoring effective syntax definitions. The engine's power, particularly its support for advanced features, enables the parsing of complex, modern language grammars that would be intractable with simpler regex engines.

  * **Core Syntax**: It supports standard features including character classes (`\d`, `\s`, `\w`, `\h`), quantifiers (`*`, `+`, `?`, `{n,m}`), anchors (`^`, `$`, `\b`, `\A`, `\z`, `\G`), and grouping (`(...)`).[7]
  * **Advanced Features**: Crucially, it provides advanced constructs that are heavily utilized in `.sublime-syntax` files:
      * **Look-arounds**: Positive and negative look-aheads (`(?=...)`, `(?!...)`) and look-behinds (`(?<=...)`, `(?<!...)`) are used to make context-sensitive matching decisions without consuming characters.[7] For example, the C syntax uses a negative lookahead (`(?!\.)`) to correctly distinguish floating-point numbers from integers.[2]
      * **Atomic Groups**: `(?>...)` is used to prevent backtracking within a group, which can be a critical performance optimization in complex patterns.[7]
      * **Named Captures and Backreferences**: Groups can be named (`(?<name>...)`) and referenced by name (`\k<name>`), improving the clarity of `captures` blocks.[8]
  * **Extended Mode (`(?x)`)**: This flag, which can be enabled within a regex, allows for insignificant whitespace and comments (`#...`). It is a best practice for complex patterns, as it dramatically improves readability and maintainability, as seen in the C and JavaScript syntaxes.[8, 2, 2]
  * **Usage in `.tmPreferences`**: The same Oniguruma engine is used for the substitution patterns (`s/old/new/g`) in `symbolTransformation` keys within `.tmPreferences` files.[2]

## II. The Syntax Highlighting State Machine

The syntax highlighting engine can be formally modeled as a non-deterministic pushdown automaton with backtracking capabilities. It operates on a stack of `contexts`, where each context represents a state. The engine reads the text token by token, and based on `match` patterns, it transitions between states by manipulating the context stack.

### 2.1. The Context Stack: A Pushdown Automaton Model

The engine's core is a stack of active `contexts`. Processing begins with the `main` context at the bottom of the stack and proceeds by pushing and popping states in response to matched text.[1, 9]

  * **`push`**: This command adds one or more new contexts to the top of the stack. The topmost context becomes the active state. This action is used for entering hierarchically nested structures, such as a `string` context being pushed upon matching an opening quote, or a `block` context being pushed upon matching an opening brace.[1, 9]
  * **`pop`**: This command removes one or more contexts from the top of the stack. `pop: true` removes a single context, while an integer `N` (`pop: <N>`) removes `N` contexts. This is the corresponding action for exiting a nested structure, such as matching a closing quote or brace.[1, 2]
  * **`set`**: This command performs an atomic `pop` followed by a `push`. It replaces the current context on the stack with a new set of one or more contexts. This is a critical optimization that allows for horizontal state transitions between sequential language elements (e.g., from a `keyword` to an `identifier`) without increasing the depth of the context stack. The design of the state machine prioritizes performance, and syntax authors should prefer `set` for sequential transitions, reserving `push`/`pop` for true hierarchical nesting. The `Android.bp` syntax masterfully demonstrates this principle by chaining `set` commands to parse entire assignment statements with a minimal stack depth.[1, 9, 2]

### 2.2. Scoping Mechanics

As the state machine processes text, it applies scope names to matched regions. These scopes are the primary output of the syntax definition.

  * **`scope`**: Assigns a single scope name to the entire text matched by a `match` pattern.[1]
  * **`captures`**: A dictionary that maps integer capture group numbers from the `match` regex to specific scope names. This allows for fine-grained scoping of different parts of a single complex token. For example, a number literal can have its base (`0x`), value, and suffix (`L`) scoped independently.[1, 2]
  * **`meta_scope`**: Applies a scope to the entire region covered by a context, from the text that triggered the `push` to the text that triggers the `pop`. This is used to define broad semantic regions, such as `meta.function.c` covering an entire C function.[1, 2]
  * **`meta_content_scope`**: Similar to `meta_scope`, but it excludes the trigger and pop tokens themselves. It applies the scope only to the content *within* the delimiters. The choice between `meta_scope` and `meta_content_scope` is a conscious design decision that determines the precise boundaries of logical code units for consumption by other editor features.[1, 2]
  * **`clear_scopes`**: Removes a specified number of scopes (or all scopes if `true`) from the current scope stack. This is essential when embedding one syntax within another to prevent the scopes from the host syntax from "bleeding" into and incorrectly styling the embedded code.[1, 2]

### 2.3. Advanced Flow Control: Handling Ambiguity

For ambiguous grammars where a token's meaning depends on subsequent tokens, the engine provides a backtracking mechanism. This transforms the state machine from a simple PDA into a more powerful parser capable of resolving local ambiguities, a pragmatic trade-off between parsing accuracy and performance.

  * **`branch_point`**: Assigns a unique identifier to a `match` pattern, marking it as a location to which the engine can rewind its parsing state.[1, 2]
  * **`branch`**: Used in conjunction with a `branch_point`, this key provides a list of alternative contexts for the engine to attempt in sequence.[1, 2]
  * **`fail`**: An action that, when triggered within a context, immediately halts the current parsing path. The engine then rewinds its state to the specified `branch_point` and attempts the next context in the `branch` list.[1, 2]

A practical implementation of this system is found in the `Android.bp` syntax.[2] The `main` context establishes a `branch_point` named `module-or-assignment` and a `branch` list containing `[assignment, module-definition]`. The engine first attempts the `assignment` path. If it is parsing a module definition like `my_module {... }`, it will fail to find an assignment operator (`=` or `+=`). A `fail` rule within the `assignment` path then triggers, causing the engine to rewind to the start of `my_module` and attempt the second branch, `module-definition`, which succeeds.

### 2.4. Reusability and Inheritance

The `.sublime-syntax` format is designed with a "Don't Repeat Yourself" (DRY) philosophy, providing several mechanisms to reduce duplication and improve maintainability.

  * **`include`**: Inlines the patterns from another named context into the current location. This is the primary mechanism for code reuse within a single syntax definition file.[1, 2, 2]
  * **`prototype`**: A special context whose patterns are implicitly included at the beginning of every other context in the file, unless explicitly disabled with `meta_include_prototype: false`. This is the ideal location for ubiquitous language elements like comments or, in Rust's case, macro metavariables.[1, 2]
  * **`extends`**: Inherits all `variables` and `contexts` from a specified parent `.sublime-syntax` file. This is a powerful architectural pattern for managing language families. The TypeScript syntax, for example, `extends` the JavaScript syntax, inheriting its entire grammar and only defining the additional or modified constructs unique to TypeScript.[1, 2] Child syntaxes can inject rules into or override parent contexts using the `meta_prepend: true` and `meta_append: true` keys.

## III. Scope Naming Conventions

Scope names are the fundamental interface between syntax definitions, color schemes, and other editor features like symbol lists and indentation rules. Adherence to a standardized, hierarchical naming convention is critical for ensuring that a syntax definition works correctly with the broader Sublime Text ecosystem.

### The Hierarchical System and Selectors

The power of the scoping system derives from its hierarchical, dot-separated naming structure.[3]

  * **Structure**: Scopes are composed of dot-separated labels, ordered from most general to most specific. For example, a flow control keyword in Rust is scoped as `keyword.control.rust`.[3]
  * **Selectors**: Other parts of the editor use "selectors" to target scopes. A selector matches a scope if it is a prefix of that scope. For example, the selector `keyword.control` will match `keyword.control.rust`, `keyword.control.c`, and `keyword.control.flow.python`.[10] This abstraction layer allows color schemes and preference files to define general rules that apply across many languages, decoupling them from specific syntax implementations.
  * **Logical Operators**: Selectors can be combined using logical operators: OR (`,` or `|`), AND (`&`), and NOT (`-`), as well as grouping with parentheses `()`. This expressiveness is heavily used in `.tmPreferences` files to target complex combinations of scopes.[10, 2]

### Scopes for Symbol Definitions (`entity.name.*`)

The `entity.name.*` scope family is reserved for the identifiers of uniquely-identifiable constructs and serves as the primary signal to the indexer that a token is a symbol definition.[5] Applying these scopes correctly is the single most important step in enabling "Go To Definition."

### User-Defined Types (Classes, Structs, Enums, etc.)

The identifier of a user-defined aggregate type must be scoped with `entity.name.<construct>.<lang>`. This allows the indexer to add the type to the global symbol list.

*   **`entity.name.class.<lang>`**: For class definitions.
    *   *Implementation Example (C++):* In C++, a rule matching an identifier that is not followed by a scope resolution operator (`::`) within a class declaration is assigned the scope `entity.name.class.c++`.[6] This precisely targets the identifier of the class.

*   **`entity.name.struct.<lang>`**: For struct definitions.
    *   *Implementation Example (C):* In C, the identifier following a `struct` keyword is assigned the scope `entity.name.struct.c`.[6]

*   **`entity.name.enum.<lang>`**: For enum definitions.
    *   *Implementation Example (Rust):* In Rust, the identifier following an `enum` keyword is assigned the scope `entity.name.enum.rust`.[6]

*   **`entity.name.union.<lang>`**: For union definitions.
    *   *Implementation Example (C):* In C, the identifier following a `union` keyword is assigned the scope `entity.name.union.c`.[6]

*   **`entity.name.trait.<lang>`**: For trait definitions (a Rust construct).
    *   *Implementation Example (Rust):* In Rust, a rule matching an identifier following the `trait` keyword uses captures to apply `entity.name.trait.rust` to the trait's identifier.[6]

*   **`entity.name.interface.<lang>`**: For interface definitions.
    *   *Implementation Example (TypeScript):* In TypeScript, the identifier for an interface is assigned the scope `entity.name.interface.js`.[6]

### Functions and Methods

The identifier of a function or method definition must be scoped with `entity.name.function.<lang>`.

*   **Base Scope:** `entity.name.function.<lang>`
    *   *Implementation Example (C):* In C, a pattern matching an identifier followed by an opening parenthesis is assigned the scope `meta.function.c entity.name.function.c`.[6] This pattern correctly identifies a token as a function name, distinguishing it from a variable declaration.

*   **Specializations:**
    *   **`entity.name.function.constructor.<lang>`**: For class constructors. The C++ syntax uses this for identifiers that match the class name and are followed by a parameter list.[6]
    *   **`entity.name.function.destructor.<lang>`**: For class destructors. The C++ syntax applies this to identifiers prefixed with a tilde (`~`).[6]
    *   **`entity.name.function.preprocessor.<lang>`**: For function-like macros. The C syntax applies this to the identifier in a `#define` directive that is followed by an opening parenthesis.[6]

### Modules and Namespaces

The identifier for a module or namespace definition should be scoped with `entity.name.module.<lang>` or `entity.name.namespace.<lang>`.

*   **`entity.name.module.<lang>`**: For module definitions.
    *   *Implementation Example (Rust):* In Rust, a rule matching an identifier following the `mod` keyword uses captures to apply `entity.name.module.rust` specifically to the identifier part of the module declaration.[6]
    *   *Implementation Example (Android.bp):* In Android.bp, the content of a string assigned to a `name` property is given the `meta_content_scope: entity.name.module.bp`.[6]

*   **`entity.name.namespace.<lang>`**: For namespace definitions.
    *   *Implementation Example (C++):* In C++, the identifier following a `namespace` keyword is scoped with `meta.namespace.c++ entity.name.namespace.c++`.[6]

### Type Aliases

An identifier that creates a new name for an existing type should be scoped as `entity.name.type.<lang>`.

*   **`entity.name.type.typedef.<lang>`**: For C-style `typedef` statements.[6]
*   **`entity.name.type.<lang>`**: For modern type aliasing, as seen with Rust's `type` keyword [6] and TypeScript's `type` keyword.[6]
*   **`entity.name.type.using.<lang>`**: For C++ `using` aliases.[6]

### Constants

Globally defined constants that should be indexed are scoped as `entity.name.constant.<lang>`. This makes them discoverable via "Go To Symbol in Project."

*   *Implementation Example (Rust):* The identifiers of `const` and `static` items in Rust are scoped with `entity.name.constant.rust`.[6]
*   *Implementation Example (C):* Enum members in C are scoped as `entity.name.constant.c`, making them globally available symbols.[6]

### Controlling Indexing: Forward Declarations

To declare a symbol for syntax highlighting purposes without adding it to the global index (preventing it from appearing in "Go To Symbol in Project"), append `.forward-decl` to the definition scope.

This mechanism is essential for languages like C and C++ that use forward declarations to inform the compiler about a type's existence without providing its full definition. These declarations should not be the target of "Go To Definition"; the full definition should be. The scope naming convention provides for this distinction.[5]

*   *Implementation Example (C):* In C, a struct name followed immediately by a semicolon is scoped as `entity.name.struct.forward-decl.c`, while a name followed by a body is scoped as `entity.name.struct.c`.[6]

This demonstrates that the scoping system allows syntax authors to control a symbol's visibility within Sublime Text's code intelligence features, mirroring the language's own semantics.

### Scopes for Symbol References

While the scopes for definitions are strict and prescriptive, the scopes for references are more conventional than functional. The indexer primarily relies on matching the text of a token to a known definition. The reference scope's main purpose is to provide appropriate syntax highlighting and to signal that the token is not a keyword or literal. This leads to an asymmetry: definition scopes are rigid, while reference scopes are flexible.

#### Referencing User-Defined Types

When a user-defined type is used (e.g., in a variable declaration, as a parameter, or as a return type), it should be given a scope that indicates its role as a type.

*   **Common Scopes:**
    *   **`support.class.<lang>`**: A widely used scope for referencing a class, struct, or other module-like type. The `Android.bp.sublime-syntax` file scopes the module type identifier with `support.class.bp`.[6]
    *   **`storage.type.<lang>`**: A general scope for any type name. This is used heavily in the Rust syntax for almost all type references.[6]
    *   **`variable.other.<lang>`**: Can be used for references, especially when the distinction between a type and a variable is ambiguous without full semantic analysis. `Android.bp.sublime-syntax` uses `variable.other.module.bp` for module references within property values.[6]

*   **Inheritance Clauses:** When a class or interface is being inherited from or implemented, its name should be scoped as `entity.other.inherited-class.<lang>`.
    *   *Implementation Example (C++):* In C++, the names of base classes in an inheritance clause are scoped with `entity.other.inherited-class.c++`.[6]

#### Function and Method Calls

The identifier of a function or method being called must be scoped as `variable.function.<lang>`. This is one of the most consistent and important conventions for references.

*   *Implementation Example (All Major Syntaxes):* For example, the syntaxes for C, C++, and JavaScript all apply `variable.function.<lang>` to the identifier being invoked in a function call.[6, 6, 6] This ensures that function calls are highlighted distinctly from variable references.

#### Referencing Modules and Namespaces

When referencing an entity from another module or namespace, the components of the path should be scoped appropriately. There is no single strict rule, but common practice is to scope them as generic types or variables.

*   **Common Scopes:**
    *   `variable.other.module.<lang>`: Used in `Android.bp.sublime-syntax` for references to other modules.[6]
    *   `support.class.<lang>` or `storage.type.<lang>`: Often, namespace and module path components are scoped as if they were types, which is a common pattern in the C++ and Rust syntaxes.[6, 6]

### The Role of Structural `meta` Scopes

`meta` scopes are used to apply a scope to a larger, logical block of code. They provide hierarchical context to the document, which is invaluable for color schemes, plugins, and editor features like folding.

It must be understood that **`meta` scopes have no direct functional effect on symbol indexing** for "Go To Definition" or "Find References." They are purely for providing structural context. Their use is optional but is a strongly recommended best practice for creating a high-quality, maintainable, and extensible syntax definition.[4, 5]

The implementation of `meta` scopes is most straightforward when the logical block has explicit start and end tokens (such as `{` and `}`) or when the language has a dedicated statement terminator (such as `;`).

### Reference Summary Table

The following table provides a quick reference for the standard scope mappings required for symbol indexing.

| Language Construct | Definition Scope (`entity.name.*`) | Common Reference Scopes |
| :--- | :--- | :--- |
| **Class** | `entity.name.class.<lang>` | `support.class.<lang>`, `entity.other.inherited-class.<lang>` |
| **Struct** | `entity.name.struct.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Enum** | `entity.name.enum.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Union** | `entity.name.union.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Trait** | `entity.name.trait.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Interface** | `entity.name.interface.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Concept** | `entity.name.concept.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Function/Method** | `entity.name.function.<lang>` | `variable.function.<lang>` |
| **Constructor** | `entity.name.function.constructor.<lang>` | `variable.function.<lang>` (as part of `new` expression) |
| **Destructor** | `entity.name.function.destructor.<lang>` | `variable.function.<lang>` |
| **Module/Namespace** | `entity.name.module.<lang>`, `entity.name.namespace.<lang>` | `variable.other.module.<lang>`, `support.class.<lang>` |
| **Type Alias** | `entity.name.type.<lang>`, `entity.name.type.typedef.<lang>`, `entity.name.type.using.<lang>` | `support.class.<lang>`, `storage.type.<lang>` |
| **Global Constant** | `entity.name.constant.<lang>` | `variable.other.constant.<lang>` |
| **Enum Member** | `entity.name.constant.<lang>` | `variable.other.constant.<lang>` |
| **Macro (Function-like)** | `entity.name.function.preprocessor.<lang>` | `variable.function.assumed-macro.<lang>` |
| **Label** | `entity.name.label.<lang>` | (Not typically scoped as a reference) |

A table showing all the known scopes.

| Category | Scope | Description |
| :--- | :--- | :--- |
| **Comments** | `comment.line.*` | A single-line comment. |
| | `comment.block.*` | A multi-line block of comments. |
| | `punctuation.definition.comment.*` | The symbols that begin and end a comment (e.g., `//`, `/*`, `*/`). |
| **Entities (Definitions)** | `entity.name.class.*` | The name of a class definition. |
| | `entity.name.struct.*` | The name of a struct definition. |
| | `entity.name.enum.*` | The name of an enum definition. |
| | `entity.name.union.*` | The name of a union definition. |
| | `entity.name.trait.*` | The name of a trait definition. |
| | `entity.name.interface.*` | The name of an interface definition. |
| | `entity.name.type.*` | The name of a type alias (e.g., from `typedef` or `using`). |
| | `entity.name.function.*` | The name of a function or method definition. |
| | `entity.name.function.constructor.*` | The name of a constructor method. |
| | `entity.name.function.destructor.*` | The name of a destructor method. |
| | `entity.name.namespace.*` | The name of a namespace or package definition. |
| | `entity.name.module.*` | The name of a module definition. |
| | `entity.name.constant.*` | The name of a declared constant (e.g., `const`, enum members). |
| | `entity.name.label.*` | The name of a label (for `goto`). |
| | `entity.name.tag.*` | The name of an HTML/XML tag. |
| | `entity.other.inherited-class.*` | The name of a class in an inheritance clause. |
| | `entity.other.attribute-name.*` | The name of an attribute in HTML/XML/JSX. |
| **Entities (Forward Declarations)** | `entity.name.*.forward-decl.*` | A forward declaration of a class, struct, etc., that should not be indexed. |
| **Keywords** | `keyword.control.*` | Control flow keywords like `if`, `for`, `while`, `return`. |
| | `keyword.operator.*` | Operators represented by symbols (`+`, `-`, `*`, `/`, `==`, `&`). |
| | `keyword.operator.word.*` | Operators represented by words (`and`, `or`, `not`). |
| | `keyword.declaration.*` | Keywords that declare a new entity (`class`, `struct`, `fn`). |
| **Storage** | `storage.type.*` | Built-in primitive types (`int`, `bool`, `char`, `void`). |
| | `storage.modifier.*` | Modifiers that affect storage or visibility (`static`, `public`, `const`, `final`). |
| **Variables & References** | `variable.parameter.*` | A variable declared as a function or method parameter. |
| | `variable.function.*` | A reference to a function or method (i.e., a function call). |
| | `variable.language.*` | Language-specific variables (`this`, `self`, `super`). |
| | `variable.other.*` | A general-purpose variable. |
| | `variable.other.member.*` | A member variable of a class or struct. |
| | `variable.other.constant.*` | A reference to a constant. |
| | `variable.annotation.*` | The name of a decorator or annotation. |
| **Constants** | `constant.numeric.*` | Numeric literals (e.g., `123`, `0.5`, `0xFF`). |
| | `constant.character.escape.*` | Escaped characters in strings (e.g., `\n`, `\t`, `\u2026`). |
| | `constant.language.*` | Built-in language constants (`true`, `false`, `null`, `nullptr`). |
| | `constant.other.placeholder.*` | Placeholders in format strings (e.g., `%s`, `%d`). |
| **Strings** | `string.quoted.single.*` | A string enclosed in single quotes. |
| | `string.quoted.double.*` | A string enclosed in double quotes. |
| | `string.quoted.raw.*` | A raw string literal where escape sequences are not processed. |
| | `punctuation.definition.string.begin.*` | The opening quote of a string. |
| | `punctuation.definition.string.end.*` | The closing quote of a string. |
| **Support (Built-ins)** | `support.function.*` | Built-in or standard library functions. |
| | `support.class.*` | Built-in or standard library classes and types. |
| | `support.constant.*` | Built-in or standard library constants. |
| | `support.type.*` | Built-in or standard library type names. |
| **Punctuation** | `punctuation.accessor.*` | Member access operators (`.`, `->`, `::`). |
| | `punctuation.separator.*` | Separators between elements (`,`, `;` in for loops). |
| | `punctuation.terminator.*` | Statement terminators (`;`). |
| | `punctuation.section.{block,braces,group,parens,brackets}.*` | Delimiters for blocks, groups, etc. (`{`, `}`, `(`, `)`, `[`, `]`). |
| **Meta (Structural)** | `meta.block.*` | A generic block of code, usually enclosed in `{}`. |
| | `meta.function.*` | The entire scope of a function definition, including name, parameters, and body. |
| | `meta.function-call.*` | The entire scope of a function call, including the name and arguments. |
| | `meta.class.*` | The entire scope of a class definition. |
| | `meta.struct.*` | The entire scope of a struct definition. |
| | `meta.enum.*` | The entire scope of an enum definition. |
| | `meta.path.*` | A fully-qualified identifier path (e.g., `namespace::class`). |
| | `meta.tag.*` | An entire HTML/XML/JSX tag from `<` to `>`. |
| | `meta.preprocessor.*` | A preprocessor directive. |
| **Invalid** | `invalid.illegal.*` | A token that is syntactically incorrect. |
| | `invalid.deprecated.*` | A token representing deprecated syntax. |

## IV. Associated Configuration Files

Beyond syntax highlighting, a complete language package requires configuration files to integrate with other editor features like symbol navigation, indentation, and word manipulation. These files bridge the gap between the semantic scopes produced by `.sublime-syntax` and concrete editor behaviors.

### 4.1. `.tmPreferences` Files

These are XML-based Property List (`plist`) files that control features based on scope selectors.[2]

  * **File Format**: The root of the file is a `<plist>` tag containing a `<dict>`. The primary keys in this dictionary are `scope` (a scope selector string) and `settings` (another dictionary containing the actual rules).[2]
  * **Indentation Rules**: The `settings` dictionary can contain keys that define auto-indentation behavior using Oniguruma regular expressions [2]:
      * `increaseIndentPattern`: A regex for lines that should cause the next line to be indented (e.g., a line ending in `{`).
      * `decreaseIndentPattern`: A regex for lines that should be de-dented (e.g., a line containing `}`).
      * `bracketIndentNextLinePattern`: A regex for control structures (`if`, `for`) that indent the next line even without braces.
      * `unIndentedLinePattern`: A regex for lines that should ignore indentation rules entirely (e.g., preprocessor directives).
  * **Symbol Lists**: The `settings` dictionary controls which scoped tokens appear in the symbol lists [2, 2]:
      * `showInSymbolList`: An integer (`1` for true) that adds text matching the file's `scope` selector to the file-local symbol list (accessible via `Ctrl+R`).
      * `showInIndexedSymbolList`: An integer (`1` for true) that adds matching text to the project-wide symbol list (accessible via `Ctrl+Shift+R`).
  * **Symbol Transformations**: To clean up symbol names before display, the `settings` dictionary can include [2]:
      * `symbolTransformation`: A semicolon-separated list of Oniguruma substitution regexes (e.g., `s/prefix_//g;`) applied to symbols in the local list.
      * `symbolIndexTransformation`: The same as above, but for the project-wide symbol list.

The complete workflow for symbol navigation involves the `.sublime-syntax` file applying a semantic scope like `entity.name.function`, which is then targeted by a `.tmPreferences` file's `scope` selector to enable its inclusion in the symbol list.

### 4.2. `.sublime-settings` Files

These are JSON files that override global editor settings on a per-syntax basis. They must be named `<SyntaxName>.sublime-settings` (e.g., `Android.bp.sublime-settings`).[2]

  * **File Format**: A simple JSON object containing key-value pairs that correspond to standard Sublime Text settings.[2]
  * **Common Use Case**: A primary use is to define `word_separators`. This string of characters tells the editor what to treat as a word boundary for actions like double-click selection and `Ctrl+Left/Right` navigation. This allows for fine-tuning the editing experience to match a language's conventions, such as treating file paths with `/` as single "words".[2]

## V. Authoring and Validation

Developing a high-quality syntax definition is an engineering discipline that requires adherence to best practices and rigorous testing.

### 5.1. Best Practices for Syntax Development

Analysis of the provided professional-grade syntax definitions reveals a set of common best practices for creating robust, maintainable, and performant files.

  * **Modularity**: Decompose the language grammar into small, semantically distinct contexts (e.g., `comments`, `strings`, `numbers`) and reuse them with `include`.[2, 2, 2]
  * **State Management**: Prefer `set` for sequential state transitions to maintain a shallow context stack and improve performance. Reserve `push`/`pop` for true hierarchical nesting (e.g., delimited blocks).[2]
  * **Robustness**: A syntax must handle invalid and incomplete code gracefully, as it is used in a live editing environment. Use the `branch`/`fail` mechanism to backtrack from incorrect parsing paths. Explicitly define scopes for illegal constructs (`invalid.illegal`) rather than allowing the parser to fail silently.[11, 2, 2]
  * **Readability**: Use the `variables` block for all non-trivial regexes. Employ the `(?x)` regex flag to add whitespace and comments to complex patterns, making them self-documenting.[2, 2]
  * **Precision**: Use `captures` to apply fine-grained scopes to sub-components of a token. Use look-arounds (`(?=...)`, `(?<=...)`) to make context-sensitive decisions without consuming text, leading to more accurate matching.[2]
  * **Convention Adherence**: Strictly follow the scope naming conventions outlined in Section III to ensure broad compatibility with color schemes, plugins, and built-in editor features.[3, 12]
  * **Prototyping**: Place ubiquitous language elements, such as comments, in the `prototype` context for implicit inclusion across the syntax definition.[2]

### 5.2. Syntax Test Files

Sublime Text includes a built-in framework for automated testing of syntax definitions, which is critical for validation and preventing regressions. Manually testing every language construct after every change is infeasible; the test framework automates this process.[1, 11]

  * **File Format**:
      * Test files must be located in the same package as the syntax file and named `syntax_test_<description>.<extension>`.[1]
      * The first line must be a comment that identifies the syntax file to be tested: `<comment_token> SYNTAX TEST "<path_to_syntax_file>"`.[1] For a C syntax test, this would be `// SYNTAX TEST "Packages/C/C.sublime-syntax"`.
  * **Running Tests**: Tests are executed via the `Build` command (`Ctrl+B`) when either the syntax file or a corresponding test file is active.[1]
  * **Scope Assertions**:
      * A caret `^` on the line immediately following a line of test code asserts the scope of the single character directly above it.[1]
      * An arrow `<-` asserts the scope of the entire region of text it covers.[1]
      * The assertion itself is a scope selector. The test passes if the selector matches the scope(s) applied to the specified text.c
      ```c
        // SYNTAX TEST "Packages/C/C.sublime-syntax"
        int main() {
        // <----- source.c
        if (true) {}
        //  <- keyword.control.c
        //     ^ constant.language.c
        }
      ```
  * **Symbol Assertions**:
      * An at-sign `@` on the line below an identifier asserts its status in the symbol list.[1]
      * The assertion is followed by a keyword indicating the expected symbol type: `none`, `definition`, `reference`, `local-definition`, or `global-definition`.
      ```c
      // SYNTAX TEST "Packages/C/C.sublime-syntax"
      void my_func();
      //   ^ entity.name.function.c
      //   @definition
      ```
### 5.3. Validate the sublime-syntax files

`.sublime-syntax` can be checked to see if it's well formed by executing the `yamllint` command.
You should do this every time you make a change to a `.sublime-syntax` file.


### 5.4 Advanced Context Management Patterns

Analysis of sophisticated syntax definitions like `Android.bp` reveals advanced patterns for managing the context stack, handling errors, and creating modular, reusable parsing logic. These techniques go beyond simple `push` and `pop` operations and are key to building robust and maintainable syntaxes. [1]

#### Error Handling with `or_invalid`

In a live editing environment, code is often in an incomplete or invalid state. A robust syntax definition must handle this gracefully without breaking highlighting for the rest of the file. A common pattern is to use a dedicated "invalid" context as a catch-all at the end of a rule list.

The `Android.bp` syntax defines an `or_invalid` context for this purpose: [1]

```yaml
or_invalid:
  # Don't be too greedy to allow recovery.
  - match: '\w'
    scope: invalid.illegal.bp
  - match: '\S'
    scope: invalid.illegal.bp
```

This context is included at the end of primary contexts like `main`. If no other rule in `main` matches, the engine attempts the rules from `or_invalid`. It will match the next single word character (`\w`) or non-whitespace character (`\S`) and apply the `invalid.illegal` scope. Crucially, this rule does *not* `pop` or `set` the context. It simply consumes the invalid token, provides visual feedback to the user, and allows the current context (`main`) to continue parsing from the next character, enabling quick recovery. [1]

#### Fallback Exits with `or_pop`

Sometimes, a context is pushed with the expectation of finding a specific kind of token (e.g., a property name). If that token isn't present, the parser can get stuck. The `or_pop` pattern provides a clean exit strategy.

Consider this context from `Android.bp`: [1]

```yaml
or_pop:
  - match: '(?=\S)'
    pop: true
```

This context is included as the final rule in contexts that expect a specific input, such as `property`. The `property` context first tries to match various valid property identifiers. If none match, it falls through to the included `or_pop` rule. The pattern `(?=\S)` is a positive lookahead that matches if there is any non-whitespace character ahead, but it doesn't consume it. The action is `pop: true`. [1]

This means if the `property` context fails to find a valid property name but the line is not empty, it will simply pop itself off the stack. Control returns to the parent context (e.g., `module-body`), which can then continue parsing from the same location, preventing the syntax from getting locked in an incorrect state. [1]

#### Self-Popping Contexts

A powerful architectural pattern for creating reusable parsing logic is the "self-popping" context. This pattern is used when a parent context needs to parse a sub-component (like a value or expression) but doesn't need to know the specific details of its grammar.

The `Android.bp` syntax uses this for its `expression` context. A parent context, such as `list-body`, will `push` the `expression` context and trust it to handle its own lifecycle. [1]

The `expression` context is designed to:

1.  Match a complete expression (e.g., a number, a string, a boolean).
2.  Upon a successful match, use `set` or `pop` to remove itself (and any helper contexts it may have pushed) from the stack.
3.  Return control to the parent context at the position immediately following the parsed expression.

**Example Trace:**
Consider parsing a list like `[123, "abc"]`.

1.  The `list` context pushes `list-body` and `expression`. The stack is `[..., list, list-body, expression]`.
2.  The `expression` context includes `number`, which matches `123` and uses `set: expression-continuation`. This pops `expression` and pushes `expression-continuation`. The stack is now `[..., list, list-body, expression-continuation]`.
3.  The `expression-continuation` context sees the comma `,` but has no rule for it. Its fallback rule `match: '(?=\S)'` with `pop: true` triggers. It pops itself. The stack is now `[..., list, list-body]`.
4.  Control returns to `list-body`. Its rule `match: ','` now triggers, and it pushes `expression` again to parse the next element.

This design decouples the list-parsing logic from the expression-parsing logic. The `list-body` only needs to know how to handle delimiters (`[`, `,`, `]`) and delegate the parsing of its contents to the reusable `expression` context. [1]


# Grammar references
Always check the `grammar/` golder for a specific language reference.
