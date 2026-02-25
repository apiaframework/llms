# Scalars and Types

Apia provides built-in scalar types for common data types and supports defining custom scalars.

## Built-in Scalar Types

Reference these by symbol in field and argument definitions:

| Symbol | Ruby Type | Input Parsing | Output Cast | Description |
|--------|-----------|---------------|-------------|-------------|
| `:string` | String | `.to_s` | `.to_s` | Text values |
| `:integer` | Integer | Parsed from string via regex `/\A-?\d+\z/` | `.to_i` | Whole numbers |
| `:boolean` | TrueClass/FalseClass | `true`, `"true"`, `"yes"`, `1`, `"1"` are true; `false`, `"false"`, `"no"`, `0`, `"0"` are false | true/false | Boolean values |
| `:decimal` | Float | Parsed from float/integer/string | `.to_f` | Decimal numbers |
| `:unix_time` | Time | `Time.at(integer)` | `.to_i` | Unix timestamps (seconds since epoch) |
| `:date` | Date | Parsed from ISO 8601 `yyyy-mm-dd` | `.strftime('%Y-%m-%d')` | Calendar dates |
| `:base64` | String | `Base64.decode64` | `Base64.encode64` | Base64-encoded binary |

## Usage

### As Field Types

```ruby
field :id, :string
field :name, :string
field :age, :integer
field :active, :boolean
field :score, :decimal
field :created_at, :unix_time
field :birth_date, :date
field :avatar, :base64
```

### As Argument Types

```ruby
argument :name, :string, required: true
argument :age, :integer
argument :active, :boolean, default: true
argument :amount, :decimal
argument :since, :unix_time
argument :date, :date
```

### Arrays

Wrap any type in brackets for arrays:

```ruby
field :tags, [:string]
field :scores, [:integer]
argument :ids, [:string]
```

## Type System

Apia has a unified type system. Anywhere a type is expected, you can use:

- **Scalars** - Built-in primitive types (`:string`, `:integer`, etc.)
- **Objects** - Custom object types (`Objects::User`)
- **Enums** - Enumeration types (`Enums::StatusEnum`)
- **Argument Sets** - For argument definitions only (`ArgumentSets::UserProperties`)
- **Polymorphs** - For field definitions, returns different types based on runtime matching

### Type Usage Rules

| Type | Usable as Field? | Usable as Argument? |
|------|-----------------|-------------------|
| Scalar | Yes | Yes |
| Enum | Yes | Yes |
| Object | Yes | No |
| ArgumentSet | No | Yes |
| Polymorph | Yes | No |

## Custom Scalars

You can define custom scalar types with parse, cast, and validation logic:

```ruby
class MoneyScalar < Apia::Scalar
  name "Money"
  description "A monetary value in cents"

  parse do |value|
    value.to_i
  end

  cast do |value|
    value.to_i
  end

  validator do |value|
    value.is_a?(Integer) && value >= 0
  end
end
```

### Custom Scalar DSL Methods

- `parse(&block)` - Convert input (from JSON/params) to the Ruby type
- `cast(&block)` - Convert the Ruby value to the output format
- `validator(&block)` - Return true if the value is valid

## Nullable Fields

By default, fields are non-nullable. Use `null: true` to indicate a field may return nil:

```ruby
field :email, :string, null: true
field :deleted_at, :unix_time, null: true
field :parent_id, :string, null: true
```
