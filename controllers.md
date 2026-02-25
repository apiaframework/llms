# Controllers

Controllers group related endpoints together and can provide shared configuration, helpers, and inline endpoint definitions.

## Defining a Controller

```ruby
module MyAPI
  module Controllers
    class TimeController < Apia::Controller

      name "Time API"
      description "Returns the current time in varying ways"

      # Register an existing endpoint class
      endpoint :now, Endpoints::TimeNowEndpoint

      # Define an inline endpoint
      endpoint :format do
        description "Format the given time"
        argument :time, type: ArgumentSets::TimeLookupArgumentSet, required: true
        field :formatted_time, type: :string
        action do
          time = request.arguments[:time]
          response.add_field :formatted_time, time.resolve.to_s
        end
      end

    end
  end
end
```

## DSL Methods

### `endpoint(name, klass = nil, &block)`

Registers an endpoint with the controller. Can reference an existing class or define one inline:

```ruby
# Reference an existing endpoint class
endpoint :now, Endpoints::TimeNowEndpoint

# Define inline with a block
endpoint :format do
  description "Format the given time"
  argument :time, type: ArgumentSets::TimeLookupArgumentSet, required: true
  field :formatted_time, type: :string
  action do
    response.add_field :formatted_time, request.arguments[:time].resolve.to_s
  end
end
```

### `authenticator(klass)`

Sets the default authenticator for all endpoints in this controller, overriding the API-level authenticator:

```ruby
class AdminController < Apia::Controller
  authenticator AdminAuthenticator

  endpoint :list, Endpoints::Admin::ListEndpoint
end
```

### `helper(name, &block)`

Defines a helper method that can be called from any endpoint in this controller using `helper(:name)`:

```ruby
class UsersController < Apia::Controller
  helper :current_user do
    User.find(request.identity.user_id)
  end

  endpoint :info do
    field :user, Objects::User
    action do
      response.add_field :user, helper(:current_user)
    end
  end
end
```

## Routing with Controllers

Controllers can be referenced in routes with endpoints identified by symbol:

```ruby
routes do
  # Direct controller + endpoint reference
  get "time/format", controller: Controllers::TimeController, endpoint: :format

  # Inside a group with default controller
  group :time do
    controller Controllers::TimeController
    get "time/now", endpoint: :now
    get "time/format", endpoint: :format
  end
end
```

## When to Use Controllers

Controllers are optional. The identity example app uses direct endpoint classes without controllers. Controllers are most useful when:

- You want to define small inline endpoints without creating separate files
- You need shared helpers across related endpoints
- You want to set a shared authenticator for a group of endpoints

For most APIs, organizing endpoints as individual classes in the `endpoints/` directory is the recommended approach.
