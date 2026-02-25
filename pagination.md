# Pagination

Apia provides built-in pagination support for list endpoints. When enabled, it automatically adds `page` and `per_page` arguments and a `pagination` field to the response.

## Enabling Pagination

Add `paginate: true` to a field definition:

```ruby
class ListEndpoint < BaseEndpoint
  name "List users"
  description "Returns a paginated list of users"

  field :users, [Objects::User], paginate: true

  def call
    users = User.order(:name)
    paginate(users)
  end
end
```

## What `paginate: true` Does

When you add `paginate: true` to a field, Apia automatically:

1. Adds an `argument :page, :integer, default: 1` with validation that it's positive
2. Adds an `argument :per_page, :integer, default: 30` with validation (positive, max 200)
3. Adds a `field :pagination, PaginationObject` to the response
4. Makes the `paginate` helper method available

## The `paginate` Helper

In your endpoint action, call `paginate(set)` with a Kaminari-compatible relation:

```ruby
def call
  users = User.order(:name)
  paginate(users)
end
```

The helper:
1. Calls `.page()` and `.per()` on the relation using the request's `page` and `per_page` arguments
2. Sets the paginated field on the response
3. Adds pagination metadata to the response

### Large Sets

For potentially large datasets, pass `potentially_large_set: true`:

```ruby
def call
  users = User.order(:name)
  paginate(users, potentially_large_set: true)
end
```

When `potentially_large_set` is true and the total count exceeds 1,000 records:
- `large_set` is set to `true` in the pagination response
- `total` and `total_pages` are omitted (set to null) to avoid expensive COUNT queries
- The query uses `.without_count` for efficiency

### Custom Maximum Per Page

```ruby
field :users, [Objects::User], paginate: { maximum_per_page: 100 }
```

## Pagination Response

The pagination field returns a `PaginationObject` with these fields:

```json
{
  "users": [...],
  "pagination": {
    "current_page": 1,
    "total_pages": 5,
    "total": 142,
    "per_page": 30,
    "large_set": false
  }
}
```

When `large_set` is true:

```json
{
  "users": [...],
  "pagination": {
    "current_page": 1,
    "total_pages": null,
    "total": null,
    "per_page": 30,
    "large_set": true
  }
}
```

## Requirements

Pagination requires the [Kaminari](https://github.com/kaminari/kaminari) gem (or a compatible pagination library that responds to `.page`, `.per`, `.total_count`, `.total_pages`, `.current_page`, `.limit_value`, and `.without_count`).

## Complete Example

```ruby
class ListEndpoint < BaseEndpoint
  name "List organizations"
  description "Returns a paginated list of organizations"

  field :organizations, [Objects::Organization], paginate: true

  scopes "organizations.read"

  def call
    organizations = Organization.accessible_by_user(request.identity.user).order(:name)
    paginate(organizations, potentially_large_set: true)
  end
end
```
