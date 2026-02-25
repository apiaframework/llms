# Argument Sets

Argument sets define reusable groups of input parameters. They are used to organize endpoint arguments into logical groups, especially for create and update operations.

## Defining an Argument Set

```ruby
module MyAPI
  module ArgumentSets
    class UserCreateProperties < Apia::ArgumentSet

      name "User creation properties"
      description "The properties required to create a user"

      argument :first_name, :string, required: true
      argument :last_name, :string, required: true
      argument :email_address, :string, required: true
      argument :time_zone, :string

    end
  end
end
```

## Using in Endpoints

```ruby
class CreateEndpoint < BaseEndpoint
  argument :properties, ArgumentSets::UserCreateProperties, required: true

  def call
    props = request.arguments[:properties].to_hash
    user = User.create!(props)
    response.add_field :user, user
  end
end
```

## Argument Options

### `required`

Marks the argument as mandatory. A `MissingArgumentError` is raised if it's not provided.

```ruby
argument :name, :string, required: true
```

### `default`

Sets a default value when the argument is not provided.

```ruby
argument :active, :boolean, default: true
argument :page, :integer, default: 1
```

### `description`

Adds a description for schema/documentation.

```ruby
argument :email, :string, required: true, description: "The user's email address"
```

### Array Arguments

Wrap the type in brackets to accept an array:

```ruby
argument :tags, [:string]
argument :role_names, [:string]
argument :networks, [:string]
```

### Nested Argument Sets

Arguments can reference other argument sets for nesting:

```ruby
argument :password, ArgumentSets::PasswordProperties
argument :totp_registrations, [ArgumentSets::TOTPRegistrationProperties]
```

### Enum Arguments

Arguments can use enum types to restrict valid values:

```ruby
argument :time_zone, Enums::TimeZoneEnum
argument :color_scheme, Enums::ColorSchemeEnum
```

### Custom Validations

Add custom validation logic with a name and block. The block receives the argument value and should return a truthy value for valid input:

```ruby
argument :age, :integer do
  validation(:must_be_positive) { |value| value.positive? }
end

argument :per_page, :integer, default: 30 do
  validation(:greater_than_zero) { |o| o.positive? }
  validation(:max_100) { |o| o <= 100 }
end
```

## Accessing Arguments

In endpoint actions, arguments are accessed through `request.arguments`:

```ruby
def call
  # Access a top-level argument
  name = request.arguments[:name]

  # Access a nested argument set, converting to a plain hash
  props = request.arguments[:properties].to_hash

  # Access individual nested arguments
  first_name = request.arguments[:properties][:first_name]

  # Check if an argument was provided (vs not sent at all)
  if request.arguments[:properties].has?(:time_zone)
    # time_zone was explicitly provided (could be nil)
  end
end
```

## Complete Examples

### Simple Properties Set

```ruby
class UserUpdateProperties < Apia::ArgumentSet
  argument :first_name, :string
  argument :last_name, :string
  argument :time_zone, :string
end
```

### Properties with Arrays and Enums

```ruby
class OrganizationCreateProperties < Apia::ArgumentSet
  argument :name, :string, required: true
  argument :identifier, :string
  argument :time_zone, Enums::TimeZoneEnum
end
```

### Complex Nested Properties

```ruby
class UserCreationProperties < Apia::ArgumentSet
  name "User creation properties"
  description "The properties required to create a user"

  argument :first_name, :string, required: true
  argument :last_name, :string, required: true
  argument :email_address, :string, required: true
  argument :time_zone, :string
  argument :password, ArgumentSets::PasswordProperties
  argument :totp_registrations, [ArgumentSets::TOTPRegistrationProperties]
  argument :recovery_codes, [ArgumentSets::RecoveryCodeProperties]
end
```

### Key-Value Argument Set

A pattern for accepting arbitrary key-value pairs:

```ruby
class KeyValue < Apia::ArgumentSet
  argument :key, :string, required: true
  argument :value, :string

  class << self
    def to_hash(arguments)
      arguments.to_h { |a| [a[:key], a[:value]] }
    end
  end
end
```

Usage in an endpoint:

```ruby
argument :metadata, [ArgumentSets::KeyValue]

def call
  metadata = ArgumentSets::KeyValue.to_hash(request.arguments[:metadata])
end
```
