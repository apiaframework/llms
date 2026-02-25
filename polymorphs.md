# Polymorphs

Polymorphs allow a single field to return different object types based on runtime conditions. This is useful when a field could contain one of several related but distinct types.

## Defining a Polymorph

```ruby
module MyAPI
  module Objects
    class EventSubject < Apia::Polymorph

      name "Event Subject"
      description "The subject of an event (could be a user, organization, or server)"

      option :user, type: Objects::User, matcher: -> (value) { value.is_a?(::User) }
      option :organization, type: Objects::Organization, matcher: -> (value) { value.is_a?(::Organization) }
      option :server, type: Objects::Server, matcher: -> (value) { value.is_a?(::Server) }

    end
  end
end
```

## DSL Methods

### `option(name, type:, matcher:)`

Adds a polymorph option. Each option defines:

- `name` - A symbol identifying this option
- `type:` - The object type (or scalar/enum) to serialize the value as
- `matcher:` - A lambda that returns true if the given value should use this option

```ruby
option :user, type: Objects::User, matcher: -> (value) { value.is_a?(::User) }
option :organization, type: Objects::Organization, matcher: -> (value) { value.is_a?(::Organization) }
```

## Using in Fields

Reference the polymorph type on a field:

```ruby
class Event < Apia::Object
  field :subject, Objects::EventSubject
end
```

## Response Format

When a polymorph field is serialized, the response includes the matched type name and the serialized value:

```json
{
  "subject": {
    "type": "user",
    "value": {
      "id": "abc-123",
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
}
```

## How It Works

1. When serializing a polymorph field, Apia iterates through the options in order
2. For each option, it calls the `matcher` lambda with the actual value
3. The first option whose matcher returns true is used
4. The value is then serialized using that option's type
5. If no option matches, an `InvalidPolymorphValueError` is raised

## Complete Example

### Define the types

```ruby
class UserObject < Apia::Object
  field :id, :string
  field :name, :string
  field :email, :string
end

class TeamObject < Apia::Object
  field :id, :string
  field :name, :string
  field :member_count, :integer
end
```

### Define the polymorph

```ruby
class Assignable < Apia::Polymorph
  name "Assignable"
  description "Something that can be assigned to a task"

  option :user, type: UserObject, matcher: -> (v) { v.is_a?(::User) }
  option :team, type: TeamObject, matcher: -> (v) { v.is_a?(::Team) }
end
```

### Use in an object

```ruby
class Task < Apia::Object
  field :id, :string
  field :title, :string
  field :assignee, Assignable, null: true
end
```
