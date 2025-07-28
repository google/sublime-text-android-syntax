# Android Init (`.rc`) File Structure

Our analysis of the C++ parser in `system/core/init` revealed the following structure:

*   **File Format**: Line-oriented plain text files (`.rc`).
*   **Tokens**: Separated by whitespace.
*   **Comments**: Lines starting with `#` are comments.
*   **Line Continuation**: A backslash (`\`) as the final character on a line merges it with the following line.
*   **Quoting**: Double quotes (`"`) can be used to treat a string with spaces as a single token.
*   **Property Expansion**: Properties are expanded using `${property.name}` or `${property.name:-default}` syntax.

*   **Top-Level Statements**: An `.rc` file is composed of three main types of sections:
    1.  `import <path>`
    2.  `on <trigger> [&& <trigger>]*`
    3.  `service <name> <pathname> [argument]*`

*   **Actions (`on` blocks)**:
    *   **Syntax**: `on <trigger> [&& <trigger>]*` followed by an indented block of commands.
    *   **Triggers**:
        *   **Event Triggers**: Simple keywords for boot milestones (e.g., `boot`, `early-init`, `fs`).
        *   **Property Triggers**: Conditions based on a system property's value (e.g., `property:ro.crypto.state=encrypted`).
    *   **Commands**: The executable statements inside an `on` block. The definitive list is in `builtins.cpp`.

*   **Services**:
    *   **Syntax**: `service <name> <pathname> [argument]*` followed by an indented block of options.
    *   **Function**: Defines a program to be launched and managed by `init`.
    *   **Options**: Modifiers for the service's behavior. The definitive list is in `service_parser.cpp`.
