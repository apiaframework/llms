# Lookup Argument Sets

Lookup argument sets are a specialized form of argument sets that provide flexible record lookups. They allow callers to find a record by any one of several unique identifiers (e.g., by ID, by slug, by email).

## Defining a Lookup Argument Set

```ruby
module MyAPI
  module ArgumentSets
    class OrganizationLookup < Apia::LookupArgumentSet

      name "Organization Lookup"
      description "Provides for organizations to be looked up"

      argument :id, type: :string
      argument :identifier, type: :string

      potential_error "OrganizationNotFound" do
        code :organization_not_found
        description "No organization was found matching any of the criteria provided in the arguments"
        http_status 404
      end

      resolver do |set, _request, scope|
        organization = if set[:id]
          scope.find_by(uuid: set[:id])
        elsif set[:identifier]
          scope.find_by(identifier: set[:identifier])
        end

        raise_error "OrganizationNotFound" if organization.nil?

        organization
      end

    end
  end
end
```

## How It Works

1. The caller provides exactly **one** of the defined arguments (e.g., `id` OR `identifier`, not both)
2. Apia validates that exactly one lookup argument is provided
3. The `resolver` block is called to find and return the actual record
4. If the record is not found, the resolver raises an appropriate error

## DSL Methods

### `argument(name, type:, **options)`

Defines a lookup argument. Each argument represents one way to look up the record.

```ruby
argument :id, type: :string
argument :email, type: :string
argument :username, type: :string
argument :sha256_fingerprint, type: :string
```

### `resolver(&block)`

Defines the logic to resolve a record from the provided arguments. The block receives:

- `set` - The argument set (hash-like, access with `set[:key]`)
- `request` - The current request object
- `scope` - An optional scope passed from the endpoint when calling `resolve(scope)`

```ruby
resolver do |set, _request, scope|
  record = if set[:id]
    scope.find_by(uuid: set[:id])
  elsif set[:email]
    scope.find_by(email: set[:email])
  end

  raise_error "NotFound" if record.nil?

  record
end
```

### `potential_error(klass_or_name, &block)`

Declares errors that the resolver may raise. These are included in the schema for any endpoint that uses this lookup.

```ruby
# Reference an existing error class
potential_error Errors::SSHKeyNotFoundError

# Define an inline error
potential_error "UserNotFound" do
  code :user_not_found
  description "No user was found matching the criteria"
  http_status 404
end
```

## Resolving in Endpoints

In an endpoint, call `.resolve()` on the argument set, optionally passing a scope (e.g., an ActiveRecord relation) to constrain the search:

```ruby
class InfoEndpoint < BaseEndpoint
  argument :organization, type: ArgumentSets::OrganizationLookup, required: true

  field :organization, Objects::Organization

  def call
    # Pass a scope to constrain the lookup to accessible records
    organization = request.arguments[:organization].resolve(
      Organization.accessible_by_user(request.identity.user)
    )
    response.add_field :organization, organization
  end
end
```

```ruby
class DeleteEndpoint < BaseEndpoint
  argument :ssh_key, ArgumentSets::SSHKeyLookup, required: true

  def call
    # Resolve within the user's SSH keys
    ssh_key = request.arguments[:ssh_key].resolve(request.identity.user.ssh_keys)
    ssh_key.destroy!
    response.add_field :status, true
  end
end
```

## Using `has?` for Checking Which Argument Was Provided

Inside the resolver, use `set.has?(:key)` to check if a specific argument was provided (more reliable than checking for nil/blank):

```ruby
resolver do |set, _request, scope|
  if set.has?(:sha1_fingerprint)
    ssh_key = scope.find_by(sha1_fingerprint: set[:sha1_fingerprint])
  elsif set.has?(:sha256_fingerprint)
    ssh_key = scope.find_by(sha256_fingerprint: set[:sha256_fingerprint])
  end

  raise_error Errors::SSHKeyNotFoundError if ssh_key.nil?

  ssh_key
end
```

## Complete Examples

### Simple Lookup (Single Field)

```ruby
class UserLookup < Apia::LookupArgumentSet
  name "User Lookup"
  description "Provides for users to be looked up"

  argument :id, type: :string

  potential_error "UserNotFound" do
    code :user_not_found
    description "No user was found matching any of the criteria provided in the arguments"
    http_status 404
  end

  resolver do |set, _request, _scope|
    user = User.find_by(uuid: set[:id])
    raise_error "UserNotFound" if user.nil?
    user
  end
end
```

### Multi-Field Lookup with Scoping

```ruby
class OrganizationLookup < Apia::LookupArgumentSet
  name "Organization Lookup"
  description "Provides for organizations to be looked up"

  argument :id, type: :string
  argument :identifier, type: :string

  potential_error "OrganizationNotFound" do
    code :organization_not_found
    description "No organization was found matching any of the criteria provided in the arguments"
    http_status 404
  end

  resolver do |set, _request, scope|
    organization = if set[:id]
      scope.find_by(uuid: set[:id])
    elsif set[:identifier]
      scope.find_by(identifier: set[:identifier])
    end

    raise_error "OrganizationNotFound" if organization.nil?

    organization
  end
end
```

### Lookup with External Error Class

```ruby
class SSHKeyLookup < Apia::LookupArgumentSet
  argument :sha1_fingerprint, :string
  argument :sha256_fingerprint, :string

  resolver do |set, _request, scope|
    if set.has?(:sha1_fingerprint)
      ssh_key = scope.find_by(sha1_fingerprint: set[:sha1_fingerprint])
    elsif set.has?(:sha256_fingerprint)
      ssh_key = scope.find_by(sha256_fingerprint: set[:sha256_fingerprint])
    end

    raise_error Errors::SSHKeyNotFoundError if ssh_key.nil?

    ssh_key
  end
end
```

## Path Parameter Resolution

When a lookup argument set is used for a path parameter (e.g., `get "users/:user"`), Apia automatically extracts the value from the URL path and passes it as the first argument in the lookup. For example, with route `get "organizations/:organization"` and a request to `/organizations/my-org`, the `OrganizationLookup` argument set receives `{id: "my-org"}` (mapped to the first declared argument).
