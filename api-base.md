# The API Base Class

The API base class is the top-level entry point for your Apia API. It inherits from `Apia::API` and defines the default authenticator, routes, scopes, and exception handling.

## Defining an API

```ruby
module MyAPI
  class Base < Apia::API

    name "My Application API"
    description "Provides access to my application's resources"

    authenticator MyAuthenticator

    exception_handler do |exception|
      # Report unhandled exceptions (e.g. to Sentry, Honeybadger)
      Sentry.capture_exception(exception)
    end

    scopes do
      add "users", "Allows access to user management"
      add "organizations", "Allows access to organization management"
    end

    routes do
      # Generate a JSON schema endpoint at /schema
      schema

      # Define routes - see routing.md for full details
      get "users", endpoint: Endpoints::Users::ListEndpoint
      post "users", endpoint: Endpoints::Users::CreateEndpoint
    end

  end
end
```

## DSL Methods

### `name(string)`

Sets a human-readable name for the API, used in schema generation.

### `description(string)`

Sets a description for the API.

### `authenticator(klass)`

Sets the default authenticator class for all endpoints. Individual endpoints or controllers can override this. See [Authentication](authentication.md).

### `exception_handler(&block)`

Registers a block that is called when an unhandled exception occurs during request processing. Use this to report errors to your error tracking service. The exception is yielded to the block.

### `scopes(&block)`

Defines the available OAuth scopes for the API. Inside the block, call `add(name, description)` to register each scope. Endpoints reference scopes by name. See [Authentication](authentication.md).

### `routes(&block)`

Defines the URL routing table. See [Routing](routing.md) for full details.

## Mounting in Rails

Apia is mounted as Rack middleware in `config/application.rb`:

```ruby
class Application < Rails::Application
  # Mount the API at /api/v1
  config.middleware.use Apia::Rack, "MyAPI::Base", "/api/v1", development: Rails.env.development?
end
```

Parameters:
- **API class** - Pass as a string (e.g. `"MyAPI::Base"`) to enable auto-reloading in development, or as a constant (e.g. `MyAPI::Base`)
- **Namespace** - The URL prefix where the API is mounted (e.g. `/api/v1`)
- **`development:`** - Set to `true` in development to enable class reloading on each request

Apia is compatible with both Rack v2.x and Rack v3.x.

## Schema Endpoint

When you call `schema` inside the routes block, Apia automatically generates a JSON schema endpoint at `GET /schema` (relative to the API namespace). This schema describes all endpoints, objects, argument sets, enums, and errors in the API.

## Validation

You can validate the entire API definition programmatically:

```ruby
errors = MyAPI::Base.validate_all
errors.each do |error|
  puts error
end
```

This checks that all types are valid, authenticators inherit from `Apia::Authenticator`, fields reference valid types, and more.
