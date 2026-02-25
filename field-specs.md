# Field Specs

Field specs control which fields are included in a response. They allow endpoints to specify a subset of fields to return from nested objects, and allow clients to request specific fields.

## Default Field Inclusion

By default, all fields on an object are included in the response. You can control the default included fields using the `include` option on a field.

### On Endpoint Fields

```ruby
class InfoEndpoint < BaseEndpoint
  # Include all top-level fields, plus specific nested fields
  field :organization, Objects::Organization, include: "*,users[user_id,owner,roles,organization_id]"
end
```

### On Object Fields

```ruby
class Time < Apia::Object
  field :time, type: Objects::Time, include: "unix,day_of_week"
end
```

## Field Spec Syntax

| Syntax | Meaning |
|--------|---------|
| `*` | All top-level fields |
| `field_name` | Include a specific field |
| `field1,field2` | Include multiple fields |
| `field[subfield1,subfield2]` | Include specific subfields of a nested object |
| `field[+extra_field]` | Include a field in addition to the defaults |
| `field[-excluded_field]` | Exclude a field from the defaults |

## Examples

### Include All Fields

```ruby
field :user, Objects::User, include: "*"
```

### Include Specific Fields Only

```ruby
field :user, Objects::User, include: "id,name,email"
```

### Nested Field Selection

```ruby
field :organization, Objects::Organization, include: "*,users[user_id,owner]"
```

This includes all top-level Organization fields and for the nested `users` field, only includes `user_id` and `owner`.

## How Field Specs Work

1. When an endpoint defines `include:` on a field, it sets the default field spec
2. During response generation, Apia checks each field against the spec
3. Fields not matching the spec are excluded from the response
4. This applies recursively to nested objects

Field conditions (defined with `condition` blocks on objects/fields) are evaluated separately from field specs. Both must pass for a field to be included.
