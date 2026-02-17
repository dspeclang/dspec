# Data Specification Schema (dspec)

A declarative language for defining data models, relationships, and constraints. Files use the `.dspec` or `.dss` extension.

## Quick Example

```dspec
/** Blog platform schema */

Model Post {
    description: "A blog post written by an author."
    table_name: "posts"

    fields {
        id:uuid() [primary_key, default: uuid()]
        title:string() [index]
        slug:string() [unique]
        body:text()
        author_id:uuid() [foreign_key: Author.id, on_delete: cascade]
        status:enum(PostStatus) [index, default: Draft]
        published_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        author: belongsTo(Author, author_id)
        comments: hasMany(Comment, post_id)
        tags: morphToMany(Tag, taggable)
    }

    indexes {
        author_status: index([author_id, status])
    }
}

Enum PostStatus:string() {
    Draft,
    Published,
    Archived
}
```

## Top-Level Declarations

A `.dspec` file contains three types of declarations:

| Declaration | Purpose                                   |
| ----------- | ----------------------------------------- |
| `Model`     | Database entity (table)                   |
| `Pivot`     | Join table for many-to-many relationships |
| `Enum`      | Enumeration with a backing type           |

## Model / Pivot

```dspec
Model Name {
    description: "Optional description."
    table_name: "custom_table_name"

    fields { ... }
    indexes { ... }
    relations { ... }
    computed_attributes { ... }
    constraints { ... }
}

Pivot Name {
    // Same sections as Model
}
```

All sections are optional. Order does not matter.

## Fields

Declared inside a `fields` block. Syntax: `name:type() [modifiers]`

```dspec
fields {
    id:uuid() [primary_key, default: uuid()]
    name:string() [index]
    code:char(6) [unique]
    description:text() [nullable]
    quantity:integer() [unsigned, default: 0]
    price:decimal(8, 2)
    status:enum(Status) [index, default: Active]
    token:string(64) [encrypted]
    created_at:timestamp() [default: now()]
    updated_at:timestamp() [default: now(), on_update: now()]
    deleted_at:timestamp() [nullable, index]
}
```

> There is no space between the field name and the colon: `name:type()`, not `name : type()`.

### Types

| Type            | Description                            | Example                  |
| --------------- | -------------------------------------- | ------------------------ |
| `uuid()`        | UUID v4                                | `id:uuid()`              |
| `string()`      | Variable-length string                 | `name:string()`          |
| `string(n)`     | String with max length                 | `token:string(312)`      |
| `char(n)`       | Fixed-length string                    | `code:char(6)`           |
| `text()`        | Unlimited text                         | `body:text()`            |
| `boolean()`     | True/false                             | `is_active:boolean()`    |
| `timestamp()`   | Date and time                          | `created_at:timestamp()` |
| `integer()`     | 32-bit integer                         | `count:integer()`        |
| `bigint()`      | 64-bit integer                         | `balance:bigint()`       |
| `decimal(p, s)` | Decimal (total digits, decimal places) | `price:decimal(8, 2)`    |
| `enum(Name)`    | Reference to an Enum declaration       | `status:enum(Status)`    |

### Modifiers

Modifiers are placed inside `[ ]` after the type.

| Modifier                   | Description                | Example                      |
| -------------------------- | -------------------------- | ---------------------------- |
| `primary_key`              | Primary key column         | `[primary_key]`              |
| `unique`                   | Unique constraint          | `[unique]`                   |
| `index`                    | Database index             | `[index]`                    |
| `nullable`                 | Allows NULL                | `[nullable]`                 |
| `encrypted`                | Value is encrypted at rest | `[encrypted]`                |
| `unsigned`                 | Unsigned numeric value     | `[unsigned]`                 |
| `default: value`           | Default value              | `[default: now()]`           |
| `foreign_key: Model.field` | Foreign key reference      | `[foreign_key: Category.id]` |
| `on_delete: action`        | FK delete behavior         | `[on_delete: cascade]`       |
| `on_update: fn()`          | Auto-update trigger        | `[on_update: now()]`         |

**`on_delete` actions:** `cascade`, `set_null`, `restrict`, `no_action`

**`default` values:** `uuid()`, `now()`, `true`, `false`, `EnumMember`, numbers, `"strings"`

Multiple modifiers are comma-separated:

```dspec
category_id:uuid() [index, foreign_key: Category.id, on_delete: cascade]
```

## Indexes

Declared inside an `indexes` block. Syntax: `name: type(columns)`

```dspec
indexes {
    // Single column
    email_idx: index(email)

    // Composite — always use brackets
    unique_name_type: unique([name, type])
    status_date: index([status, created_at])
}
```

Composite indexes **must** use `[brackets]` around the column list.

## Relations

Declared inside a `relations` block. Syntax: `name: type(args)`

```dspec
relations {
    // One-to-one / One-to-many
    category: belongsTo(Category, category_id)
    items: hasMany(Item, order_id)
    profile: hasOne(Profile, user_id)

    // Many-to-many
    roles: belongsToMany(Role, user_roles, user_id)

    // Polymorphic
    target: morphTo()
    comments: morphMany(Comment, commentable)
    image: morphOne(Image, imageable)
    tags: morphToMany(Tag, taggable)
    posts: morphedByMany(Post, taggable)
}
```

| Type                              | Arguments                               | Description               |
| --------------------------------- | --------------------------------------- | ------------------------- |
| `belongsTo(Model, fk)`            | Related model, foreign key              | Inverse of hasMany/hasOne |
| `hasMany(Model, fk)`              | Related model, foreign key              | One-to-many               |
| `hasOne(Model, fk)`               | Related model, foreign key              | One-to-one                |
| `belongsToMany(Model, pivot, fk)` | Related model, pivot table, foreign key | Many-to-many              |
| `morphTo()`                       | —                                       | Polymorphic child side    |
| `morphMany(Model, name)`          | Related model, morph name               | Polymorphic one-to-many   |
| `morphOne(Model, name)`           | Related model, morph name               | Polymorphic one-to-one    |
| `morphToMany(Model, name)`        | Related model, morph name               | Polymorphic many-to-many  |
| `morphedByMany(Model, name)`      | Related model, morph name               | Inverse of morphToMany    |

## Computed Attributes

Virtual attributes derived from field values. Not persisted in the database.

```dspec
computed_attributes {
    is_confirmed:boolean() [confirmed_at IS NOT NULL]
    is_expired:boolean() [expires_at < now()]
    is_pending:boolean() [NOT (is_confirmed OR is_expired OR is_cancelled)]
}
```

Syntax: `name:type() [expression]`

Expressions use SQL-like syntax and can reference other computed attributes.

## Constraints

Database-level check constraints.

```dspec
constraints {
    check_dates:check(start_date < end_date)
    check_amount:check(amount > 0)
    check_mutual_exclusion:check(
        (confirmed_at IS NOT NULL AND cancelled_at IS NULL)
        OR (confirmed_at IS NULL AND cancelled_at IS NOT NULL)
        OR (confirmed_at IS NULL AND cancelled_at IS NULL)
    )
}
```

Syntax: `name:check(expression)`

### Expression Operators

| Category    | Operators                       |
| ----------- | ------------------------------- |
| Comparison  | `=`, `!=`, `<`, `>`, `<=`, `>=` |
| Null checks | `IS NULL`, `IS NOT NULL`        |
| Logical     | `AND`, `OR`, `NOT`              |
| Grouping    | `( )`                           |

## Enums

Typed enumerations with a backing type.

```dspec
// String enum — values are inferred from member names
Enum Status:string() {
    Active,
    Inactive,
    Suspended
}

// String enum with explicit values
Enum Color:string() {
    Red = "red",
    Green = "green",
    Blue = "blue"
}

// Integer enum with explicit values
Enum Priority:integer() {
    Low = 1,
    Medium = 2,
    High = 3
}
```

Syntax: `Enum Name:backing_type() { Member1, Member2 = value, ... }`

Values are optional. When omitted, the member name is used as the value (suitable for string-backed enums). For non-string backing types (e.g. `integer()`), explicit values are required.

Trailing commas are allowed.

## Comments

Two styles, usable anywhere in the file:

```dspec
// Single-line comment

/* Block comment
   spanning multiple lines */

/** Documentation comment */
```

## Examples

A comprehensive examples file covering all language features is available in [EXAMPLES.md](EXAMPLES.md).

## Formal Grammar

The complete PEG grammar is available in [grammar.peg](grammar.peg).

## Editor Support

- **Visual Studio Code**: [vscode-dspec](https://github.com/dspeclang/vscode-dspec)
