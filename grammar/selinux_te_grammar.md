# SELinux Type Enforcement (.te) File Grammar

This document provides a detailed breakdown of the grammar and structure of SELinux Type Enforcement (`.te`) files, as used in Android. This is intended to be a comprehensive guide for building a robust syntax highlighter or parser.

## 1. Basic Syntax

### 1.1. Comments

All text from a `#` character to the end of the line is considered a comment.

```selinux
# This is a comment.
type adbd, domain; # This is an inline comment.
```

### 1.2. Statements

Statements are typically terminated by a semicolon `;`, though this is often omitted within M4 macro blocks. Statements can span multiple lines.

```selinux
allow domain self:process {
    fork
    sigchld
    sigkill
};
```

### 1.3. Identifiers and Conventions

Identifiers are used for types, attributes, classes, and permissions. They consist of alphanumeric characters and underscores (`_`). By convention:
*   **Types** end in `_t` (e.g., `adbd_t`, `file_type`).
*   **Attributes** often end in `_type` or describe a capability (e.g., `domain`, `mlstrustedsubject`).

### 1.4. Sets / Lists

A set or list of identifiers is enclosed in curly braces `{}`. Items are separated by whitespace.

```selinux
# A set of permissions
allow domain self:process { fork sigchld sigkill };

# A set of types
allow { domain -init } su:binder { call transfer };
```

Sets support three special operators:
*   **Negation (`-`):** Excludes an identifier from the set. This is often used with broad attributes to create exceptions.
*   **Wildcard (`*`):** Matches all identifiers of a given category (e.g., all types, all permissions).
*   **Complement (`~`):** Matches all identifiers *except* for those specified in the set.

```selinux
# Grant all permissions except open
allow servicemanager unlabeled:file ~{ open };

# Allow all domains except init and recovery
neverallow { domain -init -recovery } unlabeled:dir_file_class_set create;

# Allow access to all permissions in the file class
allow adbd adb_data_file:file *;
```

### 1.5. The `self` Keyword

The `self` keyword is a placeholder that refers to the source type of the rule. It is commonly used in the target type field to grant a domain permissions to itself.

```selinux
# Allow any domain to perform these actions on itself
allow domain self:process { fork sigchld };
```

## 2. Declarations

Declarations introduce new types, attributes, roles, classes, and permissions to the policy.

### 2.1. Type and Attribute Declarations

#### `type`
Declares a new type. It can optionally be associated with one or more previously declared attributes or have aliases.

**Syntax:**
`type <type_id> [alias <alias_id | { alias_id_set }>] [, <attribute_id_1>, <attribute_id_2>, ...];`

**Example:**
```selinux
# Declare a simple type
type adbd, domain;

# Declare a type and associate it with three attributes
type adbd_exec, exec_type, file_type, system_file_type;

# Declare a type with an alias
type bin_t alias { ls_exec_t sbin_t };
```

#### `typealias`
An alternative way to associate a previously declared type to one or more alias identifiers.

**Syntax:**
`typealias <type_id> alias <alias_id | { alias_id_set }>;`

**Example:**
```selinux
type mount_t;
typealias mount_t alias mount_ntfs_t;
```

#### `attribute`
Declares a new attribute. An attribute is a name for a group of types.

**Syntax:**
`attribute <attribute_id>;`

**Example:**
```selinux
attribute domain;
attribute file_type;
```

#### `typeattribute`
Associates a previously declared type with a previously declared attribute. This is an alternative to associating attributes at type declaration.

**Syntax:**
`typeattribute <type_id> <attribute_id>;`

**Example:**
```selinux
type setroubleshootd_t;
attribute domain;
typeattribute setroubleshootd_t domain;
```

### 2.2. Role Declarations

#### `role`
Declares a new role. Roles are part of SELinux RBAC (Role-Based Access Control).

**Syntax:**
`role <role_id> types <type_id | { type_id_set }>;`

**Example:**
```selinux
role system_r types { domain };
```

#### `attribute_role`
Declares a role attribute.

**Syntax:**
`attribute_role <attribute_id>;`

**Example:**
```selinux
attribute_role roles_for_x;
```

### 2.3. Class and Permission Declarations

#### `class`
Declares a new object class.

**Syntax:**
`class <class_id> [ inherits <common_id> ] [ { <permission_id_set> } ]`

**Example:**
```selinux
# Declare a new class
class db_tuple

# Inherit common permissions and add specific ones
class db_blob inherits database { read write import export }
```

#### `common`
Declares a set of common permissions that can be inherited by multiple classes.

**Syntax:**
`common <common_id> { <permission_id_set> }`

**Example:**
```selinux
common database { create drop getattr setattr relabelfrom relabelto }
```

### 2.4. Boolean Declaration

#### `bool`
The `bool` statement is used to specify a boolean identifier and its initial state (true or false) that can then be used with the `if` statement to form a 'conditional policy'.

**Syntax:**
`bool <bool_id> <default_value>;`

**Example:**
```selinux
bool allow_execheap false;
```

## 3. Access Control Rules

These rules define the core of the Type Enforcement policy.

### 3.1. `allow`
Grants a permission from a source type (subject) to a target type (object) for a given set of object classes. This is the most common rule in any policy.

**Syntax:**
`allow <source_types> <target_types> : <object_classes> <permissions>;`

**Example:**
```selinux
allow adbd init:fd use;
allow domain self:chr_file { read write };
```

### 3.2. `neverallow`
Asserts that a specific permission must **never** be granted by any `allow` rule in the final, compiled policy. This is a compile-time check, not a runtime denial.

**Syntax:**
`neverallow <source_types> <target_types> : <object_classes> <permissions>;`

**Example:**
```selinux
neverallow { domain -init } unlabeled:dir_file_class_set create;
```

### 3.3. `dontaudit`
Prevents the kernel from logging a denial message when a specific operation is denied. This is used to quiet noisy, expected denials.

**Syntax:**
`dontaudit <source_types> <target_types> : <object_classes> <permissions>;`

**Example:**
```selinux
dontaudit domain postinstall_mnt_dir:dir audit_access;
```

### 3.4. `auditallow`
Forces the kernel to log a message when a specific operation is allowed.

**Syntax:**
`auditallow <source_types> <target_types> : <object_classes> <permissions>;`

**Example:**
```selinux
auditallow domain su_exec:file execute;
```

### 3.5. Extended Permissions (`allowxperm`, `dontauditxperm`, `auditallowxperm`, `neverallowxperm`)
These rules handle permissions for operations, like `ioctl`, that have commands multiplexed on a single permission.

**Syntax:**
`{allowxperm|...} <source_types> <target_types> : <object_class> <operation> <commands>;`

**Example:**
```selinux
allowxperm domain self:tcp_socket ioctl { unpriv_sock_ioctls };
```

## 4. Type Transition and Change Rules

These rules govern how security contexts change for new processes and objects.

### 4.1. `type_transition`
Specifies a default type for a new object created in a specific context. An `allow` rule is still required for the operation to succeed.

**Syntax:**
`type_transition <source_type> <target_type> : <object_class> <default_new_type> [<object_name>];`

**Example:**
```selinux
type_transition initrc_t acct_exec_t:process acct_t;
type_transition unconfined_t etc_t : file system_conf_t eric;
```

### 4.2. `type_change`
Specifies a default type when relabeling an existing object.

**Syntax:**
`type_change <source_type> <target_type> : <class> <new_type>;`

**Example:**
```selinux
type_change auditadm_t sysadm_devpts_t:chr_file auditadm_devpts_t;
```

### 4.3. `type_member`
Specifies a default type when creating a polyinstantiated object.

**Syntax:**
`type_member <source_type> <target_type> : <class> <member_type>;`

**Example:**
```selinux
type_member sysadm_t user_home_dir_t:dir user_home_dir_t;
```

### 4.4. `typebounds`
Constrains the types that a domain can transition to.

**Syntax:**
`typebounds <bounding_domain> <bounded_domain>;`

**Example:**
```selinux
typebounds appdomain untrusted_app;
```

### 4.5. `role_transition`
Specifies a default role for a new object created in a specific context.

**Syntax:**
`role_transition <source_role> <target_type> : <class> <new_role>;`

**Example:**
```selinux
role_transition system_r acct_exec_t:process user_r;
```

### 4.6. `range_transition`
Specifies the MLS range for a new process or object.

**Syntax:**
`range_transition <source_type> <target_type> : <class> <new_range>;`

**Example:**
```selinux
range_transition initrc_t auditd_exec_t:process s15:c0.c255;
```

## 5. Constraint and Conditional Policies

### 5.1. `if`
Conditionally includes a block of rules based on the state of a boolean.

**Syntax:**
`if (conditional_expression) { true_list } [ else { false_list } ]`

**Example:**
```selinux
bool allow_execmem false;
if (allow_execmem) {
    allow sysadm_t self:process execmem;
}
```

### 5.2. `constrain`
Adds a constraint to a set of permissions based on user, role, or type.

**Syntax:**
`constrain <class | { class_set }> <permission | { perm_set }> <expression>;`

**Example:**
```selinux
constrain process transition ( r1 == r2 );
```

### 5.3. `validatetrans`
Controls the ability to change an object's security context based on a constraint.

**Syntax:**
`validatetrans <class | { class_set }> <expression>;`

**Example:**
```selinux
validatetrans { file } ( t1 == unconfined_t );
```

## 6. MLS Policy Statements

### 6.1. `sensitivity`
Defines an MLS sensitivity level.

**Syntax:**
`sensitivity <sensitivity_id> [alias <alias_id_set>];`

**Example:**
```selinux
sensitivity s0;
```

### 6.2. `dominance`
Defines the hierarchical relationship between sensitivity levels.

**Syntax:**
`dominance { <sensitivity_id_1> <sensitivity_id_2> ... }`

**Example:**
```selinux
dominance { s0 s1 s2 s3 }
```

### 6.3. `category`
Defines an MLS category.

**Syntax:**
`category <category_id> [alias <alias_id_set>];`

**Example:**
```selinux
category c0;
```

### 6.4. `level`
Combines a sensitivity and a set of categories into a security level.

**Syntax:**
`level <sensitivity_id>[:<category_id_set>];`

**Example:**
```selinux
level s0:c0.c255;
```

### 6.5. `mlsconstrain` and `mlsvalidatetrans`
These are the MLS-aware versions of `constrain` and `validatetrans`, which can also use security levels (`l1`, `h1`, `l2`, `h2`) in their expressions.

**Syntax:**
`mlsconstrain <class_set> <perm_set> <expression>;`
`mlsvalidatetrans <class_set> <expression>;`

**Example:**
```selinux
mlsconstrain dir search ( l1 dom l2 );
```

## 7. Other Statements

### 7.1. `permissive`
Places a specific domain in permissive mode, where denials are logged but not enforced.

**Syntax:**
`permissive <domain_type_id>;`

**Example:**
```selinux
permissive unconfined_t;
```

### 7.2. `sid`
Defines the initial Security IDs (SIDs) for the system.

**Syntax:**
`sid <name> <context>`

**Example:**
```selinux
sid kernel u:r:kernel:s0
```

### 7.3. `policycap`
Defines a policy capability, enabling or disabling certain policy features.

**Syntax:**
`policycap <capability_id>;`

**Example:**
```selinux
policycap network_peer_controls;
```

### 7.4. Object Labeling Statements

#### `fs_use_xattr`, `fs_use_task`, `fs_use_trans`
Specifies the labeling behavior for a filesystem.

**Syntax:**
`fs_use_xattr <fs_type> <context>;`
`fs_use_task <fs_type> <context>;`
`fs_use_trans <fs_type> <context>;`

#### `genfscon`
Specifies the security context for files in a filesystem that doesn't support extended attributes.

**Syntax:**
`genfscon <fs_type> <path> <context>;`

**Example:**
```selinux
genfscon proc / u:object_r:proc_t:s0
```

#### `portcon`, `netifcon`, `nodecon`
Specifies security contexts for network ports, interfaces, and nodes.

**Syntax:**
`portcon <proto> <port> <context>;`
`netifcon <interface> <context>;`
`nodecon <addr> <mask> <context>;`

#### `iomemcon`, `ioportcon`, `pcidevicecon`
Specifies security contexts for hardware objects.

**Syntax:**
`iomemcon <address> <context>;`
`ioportcon <port> <context>;`
`pcidevicecon <device> <context>;`

## 8. M4 Preprocessor Macros

**Crucially, Android's SELinux policy files are not written in pure SELinux Kernel Policy Language.** They are first processed by an M4 macro preprocessor. This means that many constructs that appear to be keywords are actually macros that expand into one or more core policy statements.

### 8.1. Macro Invocation

Macros are invoked like functions.

**Syntax:**
`<macro_name>(<arg1>, <arg2>, ...)`

**Example:**
```selinux
r_dir_file(domain, self)
domain_auto_trans({ domain -su }, crash_dump_exec, crash_dump);
```

### 8.2. Conditional Macros

These macros conditionally include blocks of policy based on build-time flags. The body is enclosed in backticks.

**Syntax:**
``<macro_name>(`<policy_rules>`)``

**Example:**
```selinux
userdebug_or_eng(`
  allow domain su:fd use;
')
```

### 8.3. Macro Definition (`define`)

While less common in `.te` files themselves (often defined in separate `.m4` or `*_macros` files), macro definitions use the `define` keyword.

**Syntax:**
``define(`<macro_name>`, `<macro_body>`)``

**Example:**
```selinux
define(`dac_override_allowed', `{
  apexd
  artd
}`)
```
