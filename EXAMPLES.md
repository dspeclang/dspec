# dspec — Examples

Practical examples covering all language features and common data modeling scenarios.

## 1. Basic Model

A simple model with common field types, modifiers, and soft deletes.

```dspec
Model Category {
    description: "Product categories organized in a tree structure."
    table_name: "categories"

    fields {
        id:uuid() [primary_key, default: uuid()]
        parent_id:uuid() [nullable, foreign_key: Category.id, on_delete: set_null] // Self-referencing FK
        name:string() [index]
        slug:string() [unique]
        description:text() [nullable]
        sort_order:integer() [unsigned, default: 0]
        is_active:boolean() [index, default: true]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
        deleted_at:timestamp() [nullable, index] // Soft delete
    }

    relations {
        parent: belongsTo(Category, parent_id)
        children: hasMany(Category, parent_id)
        products: hasMany(Product, category_id)
    }

    indexes {
        parent_sort: index([parent_id, sort_order])
    }
}
```

## 2. Enums — String, Integer, and Explicit Values

```dspec
// Implicit string values: member names are used as values
Enum OrderStatus:string() {
    Pending,
    Confirmed,
    Processing,
    Shipped,
    Delivered,
    Cancelled,
    Refunded
}

// Explicit string values: useful for snake_case or custom mappings
Enum PaymentMethod:string() {
    CreditCard = "credit_card",
    DebitCard = "debit_card",
    BankTransfer = "bank_transfer",
    DigitalWallet = "digital_wallet"
}

// Integer-backed enum: explicit values required
Enum Priority:integer() {
    Low = 1,
    Medium = 2,
    High = 3,
    Critical = 4
}

// Mixed: some members with explicit values, some without
Enum Visibility:string() {
    Public,
    Private,
    Unlisted = "unlisted_hidden"
}
```

## 3. Foreign Keys and On Delete Behaviors

Demonstrates multiple foreign keys with different `on_delete` strategies.

```dspec
Model Order {
    description: "Represents a customer purchase order."

    fields {
        id:uuid() [primary_key, default: uuid()]
        order_number:string() [unique]
        customer_id:uuid() [index, foreign_key: Customer.id, on_delete: cascade]
        assigned_to:uuid() [nullable, index, foreign_key: Staff.id, on_delete: set_null]
        status:enum(OrderStatus) [index, default: Pending]
        payment_method:enum(PaymentMethod)
        subtotal:decimal(10, 2) [unsigned]
        tax:decimal(10, 2) [unsigned, default: 0]
        total:decimal(10, 2) [unsigned]
        notes:text() [nullable]
        placed_at:timestamp() [default: now()]
        confirmed_at:timestamp() [nullable]
        shipped_at:timestamp() [nullable]
        delivered_at:timestamp() [nullable]
        cancelled_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        customer: belongsTo(Customer, customer_id)
        assigned_staff: belongsTo(Staff, assigned_to)
        items: hasMany(OrderItem, order_id)
        shipment: hasOne(Shipment, order_id)
    }

    computed_attributes {
        is_confirmed:boolean() [confirmed_at IS NOT NULL]
        is_shipped:boolean() [shipped_at IS NOT NULL]
        is_delivered:boolean() [delivered_at IS NOT NULL]
        is_cancelled:boolean() [cancelled_at IS NOT NULL]
        is_fulfillable:boolean() [is_confirmed AND NOT is_cancelled AND NOT is_shipped]
    }

    constraints {
        check_total:check(total >= subtotal)
        check_positive_amounts:check(subtotal >= 0 AND tax >= 0 AND total >= 0)
    }

    indexes {
        customer_status: index([customer_id, status])
        status_date: index([status, placed_at])
    }
}
```

## 4. Pivot Table (Many-to-Many)

A `Pivot` declaration for the join table linking products and tags.

```dspec
Model Product {
    fields {
        id:uuid() [primary_key, default: uuid()]
        name:string() [index]
        sku:string() [unique]
        category_id:uuid() [index, foreign_key: Category.id, on_delete: cascade]
        price:decimal(10, 2) [unsigned]
        stock:integer() [unsigned, default: 0]
        is_active:boolean() [default: true]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        category: belongsTo(Category, category_id)
        tags: belongsToMany(Tag, product_tags, product_id)
    }
}

Model Tag {
    fields {
        id:uuid() [primary_key, default: uuid()]
        name:string() [unique]
        slug:string() [unique]
        created_at:timestamp() [default: now()]
    }

    relations {
        products: belongsToMany(Product, product_tags, tag_id)
    }
}

/* Pivot table linking products and tags */
Pivot ProductTag {
    table_name: "product_tags"

    fields {
        product_id:uuid() [foreign_key: Product.id, on_delete: cascade]
        tag_id:uuid() [foreign_key: Tag.id, on_delete: cascade]
        created_at:timestamp() [default: now()]
    }

    indexes {
        unique_product_tag: unique([product_id, tag_id])
    }
}
```

## 5. Polymorphic Relations

Comments and media can be attached to multiple different models using polymorphic relationships.

```dspec
Model Comment {
    fields {
        id:uuid() [primary_key, default: uuid()]
        commentable_id:uuid()
        commentable_type:string()
        author_id:uuid() [foreign_key: User.id, on_delete: cascade]
        body:text()
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        commentable: morphTo() // Can belong to Post, Product, etc.
        author: belongsTo(User, author_id)
    }

    indexes {
        commentable_idx: index([commentable_id, commentable_type])
    }
}

Model Media {
    description: "Polymorphic media attachments for any model."

    fields {
        id:uuid() [primary_key, default: uuid()]
        mediable_id:uuid()
        mediable_type:string()
        file_path:string()
        file_name:string()
        mime_type:string()
        size_bytes:bigint() [unsigned]
        sort_order:integer() [unsigned, default: 0]
        created_at:timestamp() [default: now()]
    }

    relations {
        mediable: morphTo()
    }

    indexes {
        mediable_idx: index([mediable_id, mediable_type])
        mediable_sort: index([mediable_id, mediable_type, sort_order])
    }
}

// A model that HAS polymorphic relations (the other side)
Model Article {
    fields {
        id:uuid() [primary_key, default: uuid()]
        title:string() [index]
        body:text()
        author_id:uuid() [foreign_key: User.id, on_delete: cascade]
        published_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        author: belongsTo(User, author_id)
        comments: morphMany(Comment, commentable)
        featured_image: morphOne(Media, mediable)
        media: morphMany(Media, mediable)
    }
}
```

## 6. Computed Attributes

Virtual attributes derived from field values. Can reference other computed attributes for composition.

```dspec
Model Subscription {
    fields {
        id:uuid() [primary_key, default: uuid()]
        user_id:uuid() [foreign_key: User.id, on_delete: cascade]
        plan_id:uuid() [foreign_key: Plan.id, on_delete: cascade]
        starts_at:timestamp()
        ends_at:timestamp() [nullable]
        trial_ends_at:timestamp() [nullable]
        cancelled_at:timestamp() [nullable]
        paused_at:timestamp() [nullable]
        resumed_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    computed_attributes {
        // Simple null checks
        is_cancelled:boolean() [cancelled_at IS NOT NULL]
        is_paused:boolean() [paused_at IS NOT NULL AND resumed_at IS NULL]
        has_ended:boolean() [ends_at IS NOT NULL AND ends_at < now()]
        is_on_trial:boolean() [trial_ends_at IS NOT NULL AND trial_ends_at > now()]

        // Composed from other computed attributes
        is_active:boolean() [NOT is_cancelled AND NOT is_paused AND NOT has_ended]
        is_usable:boolean() [is_active OR is_on_trial]
    }

    constraints {
        check_dates:check(ends_at IS NULL OR starts_at < ends_at)
        check_trial:check(trial_ends_at IS NULL OR starts_at < trial_ends_at)
    }

    indexes {
        user_active: index([user_id, starts_at])
    }
}
```

## 7. Constraints — Mutual Exclusion and Business Rules

```dspec
Model Invitation {
    description: "An invitation that can be accepted, declined, or revoked, but only one at a time."

    fields {
        id:uuid() [primary_key, default: uuid()]
        sender_id:uuid() [index, foreign_key: User.id, on_delete: cascade]
        recipient_id:uuid() [index, foreign_key: User.id, on_delete: cascade]
        role_id:uuid() [foreign_key: Role.id, on_delete: cascade]
        message:text() [nullable]
        accepted_at:timestamp() [nullable]
        declined_at:timestamp() [nullable]
        revoked_at:timestamp() [nullable]
        expires_at:timestamp()
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    computed_attributes {
        is_accepted:boolean() [accepted_at IS NOT NULL]
        is_declined:boolean() [declined_at IS NOT NULL]
        is_revoked:boolean() [revoked_at IS NOT NULL]
        is_expired:boolean() [expires_at < now()]
        is_pending:boolean() [NOT (is_accepted OR is_declined OR is_revoked OR is_expired)]
    }

    constraints {
        // An invitation cannot be both accepted and declined
        check_mutual_exclusion:check(
            (accepted_at IS NOT NULL AND declined_at IS NULL)
            OR (accepted_at IS NULL AND declined_at IS NOT NULL)
            OR (accepted_at IS NULL AND declined_at IS NULL)
        )
        // A user cannot invite themselves
        check_self_invite:check(sender_id != recipient_id)
    }

    relations {
        sender: belongsTo(User, sender_id)
        recipient: belongsTo(User, recipient_id)
        role: belongsTo(Role, role_id)
    }

    indexes {
        recipient_status: index([recipient_id, created_at])
        sender_status: index([sender_id, created_at])
    }
}
```

## 8. Encrypted Fields and Sensitive Data

```dspec
Model ApiKey {
    description: "API keys for external integrations. The key value is encrypted at rest."

    fields {
        id:uuid() [primary_key, default: uuid()]
        name:string()
        key_hash:char(64) [unique] // SHA-256 hash for lookups
        key_prefix:char(8) // Visible prefix for identification
        secret:string(512) [encrypted] // Full key, encrypted at rest
        owner_id:uuid() [index, foreign_key: User.id, on_delete: cascade]
        scopes:text() [nullable]
        last_used_at:timestamp() [nullable]
        expires_at:timestamp() [nullable]
        revoked_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    computed_attributes {
        is_revoked:boolean() [revoked_at IS NOT NULL]
        is_expired:boolean() [expires_at IS NOT NULL AND expires_at < now()]
        is_valid:boolean() [NOT is_revoked AND NOT is_expired]
    }

    indexes {
        owner_keys: index([owner_id, created_at])
    }
}
```

## 9. Precision Numeric Fields

Financial and scientific data requiring exact precision.

```dspec
Model Transaction {
    fields {
        id:uuid() [primary_key, default: uuid()]
        account_id:uuid() [foreign_key: Account.id, on_delete: cascade]
        type:enum(TransactionType)
        amount:decimal(16, 4) [unsigned]
        currency:char(3) // ISO 4217 (USD, BRL, EUR)
        exchange_rate:decimal(12, 6) [nullable]
        description:string() [nullable]
        reference:string() [unique]
        processed_at:timestamp() [nullable]
        created_at:timestamp() [default: now()]
    }

    constraints {
        check_amount:check(amount > 0)
    }

    indexes {
        account_date: index([account_id, created_at])
    }
}

Enum TransactionType:string() {
    Credit,
    Debit,
    Transfer,
    Reversal
}
```

## 10. Self-Referencing and Complex Indexes

Hierarchical data with self-referencing foreign keys and multiple composite indexes.

```dspec
Model Employee {
    fields {
        id:uuid() [primary_key, default: uuid()]
        manager_id:uuid() [nullable, foreign_key: Employee.id, on_delete: set_null]
        department_id:uuid() [foreign_key: Department.id, on_delete: cascade]
        employee_number:string() [unique]
        first_name:string()
        last_name:string()
        email:string() [unique]
        title:string() [nullable]
        hire_date:timestamp()
        termination_date:timestamp() [nullable]
        salary:decimal(10, 2) [unsigned]
        is_active:boolean() [default: true]
        created_at:timestamp() [default: now()]
        updated_at:timestamp() [default: now(), on_update: now()]
    }

    relations {
        manager: belongsTo(Employee, manager_id)
        direct_reports: hasMany(Employee, manager_id)
        department: belongsTo(Department, department_id)
    }

    computed_attributes {
        is_terminated:boolean() [termination_date IS NOT NULL]
    }

    constraints {
        check_self_manage:check(manager_id != id)
        check_dates:check(termination_date IS NULL OR hire_date < termination_date)
        check_salary:check(salary >= 0)
    }

    indexes {
        dept_active: index([department_id, is_active])
        manager_reports: index([manager_id, is_active])
        name_search: index([last_name, first_name])
    }
}
```
