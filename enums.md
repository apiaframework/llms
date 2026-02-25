# Enums

Enums define a fixed set of allowed string values. They are used for both input arguments and output fields.

## Defining an Enum

```ruby
module MyAPI
  module Enums
    class ColorSchemeEnum < Apia::Enum

      name "Color scheme"
      description "The available color schemes for the user interface"

      value "light", "Light mode"
      value "dark", "Dark mode"
      value "system", "Follow system preference"

    end
  end
end
```

## DSL Methods

### `value(name, description = nil)`

Adds an allowed value to the enum. The name is the string value, and the description is optional documentation.

```ruby
# With description
value "pending", "The invite is pending"
value "accepted", "The invite has been accepted"

# Without description
value "light"
value "dark"

# Symbol values (converted to strings)
value :pending
value :accepted
value :rejected
value :expired
```

### `cast(&block)`

Defines custom logic to transform the value before returning it in a response:

```ruby
class InviteStateEnum < Apia::Enum
  value :pending
  value :accepted
  value :rejected
  value :expired

  cast do |value|
    value.to_s
  end
end
```

## Generating Values Dynamically

You can generate enum values from model constants or database records:

```ruby
class TimeZoneEnum < Apia::Enum
  TimeZone::VALID_IDENTIFIERS.each do |time_zone|
    value time_zone
  end
end
```

```ruby
class ColorSchemeEnum < Apia::Enum
  User::COLOR_SCHEMES.each do |scheme|
    value scheme, scheme.capitalize
  end
end
```

```ruby
class SuspensionTypeEnum < Apia::Enum
  Organization::SUSPENSION_TYPES.each_value do |type|
    value type, type
  end
end
```

## Using Enums

### As Argument Types

```ruby
argument :time_zone, Enums::TimeZoneEnum
argument :color_scheme, Enums::ColorSchemeEnum
```

### As Field Types

```ruby
field :state, Enums::InviteStateEnum
field :identity_check_state, Enums::IdentityCheckStateEnum
```

### As Object Field Types

```ruby
class Time < Apia::Object
  field :day_of_week, Objects::DayEnum do
    backend { |t| t.strftime('%A') }
  end
end
```

## Validation

When an enum is used as an argument type, Apia automatically validates that the provided value is one of the allowed values. If not, an `InvalidArgumentError` is raised with the issue `invalid_enum_value`.

## Complete Example

```ruby
class DayEnum < Apia::Enum
  value "Sunday"
  value "Monday"
  value "Tuesday"
  value "Wednesday"
  value "Thursday"
  value "Friday"
  value "Saturday"
end
```

Enums can be placed either in the `enums/` directory or alongside objects in the `objects/` directory, depending on your project conventions.
