# Objects

Objects define the shape of response data. They describe which fields are returned when an object is serialized in an API response.

## Defining an Object

```ruby
module MyAPI
  module Objects
    class User < Apia::Object

      field :id, :string, backend: :uuid
      field :name, :string
      field :first_name, :string
      field :last_name, :string
      field :email_address, :string
      field :created_at, :unix_time
      field :two_factor_auth_enabled, :boolean

    end
  end
end
```

## Field Options

### Basic Fields

```ruby
# Simple scalar field - reads the method of the same name from the source object
field :name, :string

# Nullable field - indicates the field may return nil
field :email, :string, null: true

# Array field - returns an array of values
field :tags, [:string]

# Nested object field
field :organization, Objects::Organization

# Array of objects
field :users, [Objects::User]

# Enum field
field :state, Enums::InviteStateEnum
```

### `backend` Option

The `backend` option controls how the value is extracted from the source object. By default, Apia calls the method matching the field name on the source object.

```ruby
# Symbol backend - calls a different method on the source object
field :id, :string, backend: :uuid

# Lambda backend - compute the value from the source object
field :id, :string, backend: -> (client) { client.client_id }
field :parent_id, :string, null: true, backend: -> (org) { org.parent&.uuid }
field :redirect_uris, [:string], backend: -> (client) { client.redirect_uri_list }

# Block backend using the DSL
field :full_name, :string do
  backend { |user| "#{user.first_name} #{user.last_name}" }
end

# Short-form block backend using method reference
field :unix, :integer do
  backend(&:to_i)
end

# Backend with complex logic
field :networks, [:string] do
  backend { |o| o.networks_array.map { |n| "#{n}/#{n.prefix}" } }
end
```

### `condition` Option

Conditions control whether a field is included in the response. The block receives the source object and the request.

```ruby
field :email_address, :string, null: true do
  backend { |o| o.primary_email_address&.email_address }
  condition do |_, request|
    request&.identity&.scopes&.include?("user.email")
  end
end

# Condition based on the source object
field :users, [Objects::OrganizationUser] do
  backend { |o| o.organization_users.to_a }
  condition do |organization, request|
    organization.owners.include?(request.identity.try(:user))
  end
end
```

### `description` Option

```ruby
field :managed, :boolean do
  description "Is this organization managed by another organization?"
  backend { |o| o.managed? }
end
```

### `include` Option

Controls the default field spec for nested objects. See [Field Specs](field-specs.md).

```ruby
field :organization, Objects::Organization, include: "*,users[user_id,owner]"
```

## Object Conditions

You can add conditions at the object level that determine whether the entire object should be included in the response:

```ruby
class AdminDetails < Apia::Object
  condition do |_, request|
    request.identity.admin?
  end

  field :internal_id, :integer
  field :created_by, :string
end
```

## Complete Examples

### Simple Object

```ruby
class Role < Apia::Object
  field :name, :string
  field :label, :string
  field :description, :string, null: true
  field :require_two_factor_auth, :boolean
  field :organization_id, :string, null: true do
    backend { |o| o.organization&.uuid }
  end
end
```

### Object with Nested References

```ruby
class Invite < Apia::Object
  field :id, :string, backend: :uuid
  field :inviter, Objects::User
  field :email_address, :string
  field :expires_at, :unix_time
  field :accepted_at, :unix_time, null: true
  field :rejected_at, :unix_time, null: true
  field :state, Enums::InviteStateEnum
  field :owner, :boolean, null: true
  field :role_names, [:string], null: true
end
```

### Object with Lambda Backends

```ruby
class OrganizationUser < Apia::Object
  field :user_id, :string, backend: -> (ou) { ou.user.uuid }
  field :organization_id, :string, backend: -> (ou) { ou.organization.uuid }
  field :organization_name, :string, backend: -> (ou) { ou.organization.name }
  field :owner, :boolean
  field :roles, [:string], null: true do
    backend { |ou| ou.user.roles.for_organization(ou.organization).pluck(:name) }
    description "Role names the user has been granted."
  end
end
```

### Object with Conditional Fields

```ruby
class Organization < Apia::Object
  field :id, :string, backend: :uuid
  field :name, :string
  field :identifier, :string
  field :time_zone, :string, null: true
  field :updated_at, :unix_time

  field :managed, :boolean do
    description "Is this organization managed by another organization?"
    backend { |o| o.managed? }
  end

  field :parent_id, :string, null: true do
    backend { |o| o.parent&.uuid }
  end

  # This field only appears if the authenticated user is an owner
  field :users, [Objects::OrganizationUser] do
    backend { |o| o.organization_users.to_a }
    condition do |o, req|
      o.owners.include?(req.identity.try(:user))
    end
  end
end
```
