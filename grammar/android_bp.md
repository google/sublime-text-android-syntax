# Android Blueprint (`.bp`) File Structure

Our analysis of the Go parser revealed the following structure:
*   **Statement Termination:** Statements are not terminated by newlines or semicolons. The end of a statement is inferred grammatically.
*   **Whitespace:** Newlines and other whitespace are not semantically significant.

*   **Top-Level Definitions:**
    1.  **Variable Assignment:** `variable_name = "value"` or `variable_name += ["value"]`. Can span multiple lines.
    2.  **Module Definition:** `module_type { ... }` or the legacy `module_type(...)`.

*   **Data Types & Literals:**
    *   **Strings:** `"..."` (double-quoted) and `...` (raw).
    *   **Booleans:** `true`, `false`.
    *   **Integers:** `123`, `-45`.
    *   **Lists:** `["a", "b", "c"]`.
    *   **Maps:** `{ key1: "value", key2: 123 }`.
    *   **Tuples:** Tuples (`(...)`) are **not a general-purpose data type**. They are only valid in specific parser contexts (legacy module definitions and `select` expressions).

*   **Maps vs. `select` Patterns (Crucial Distinction):**
    *   **Standard Map Keys:** In a module body or a variable assignment, map keys **must be unquoted identifiers**. Example: `{ foo: 1, bar: 2 }`.
    *   **`select` Expression "Keys":** The `{...}` block inside a `select` expression is **not a true map**. Its keys are called "patterns," and they **must be quoted strings, integers, booleans, or the keywords `default` or `any`**.

*   **Comments:** `//` for line comments and `/* ... */` for block comments. `#` is invalid.
