# Apia Framework - LLM Documentation

Apia is a Ruby framework for building strongly-typed, self-documenting HTTP APIs. It is designed to be mounted as Rack middleware in Rails applications and provides a rich DSL for defining endpoints, objects, argument sets, authentication, and more.

## Quick Start

Add Apia to your Gemfile:

```ruby
gem "apia", "~> 3.4"
```

Mount it as Rack middleware in `config/application.rb`:

```ruby
config.middleware.use Apia::Rack, "MyAPI::Base", "/api/v1", development: Rails.env.development?
```

Define your API base class:

```ruby
module MyAPI
  class Base < Apia::API
    name "My API"
    description "My application's API"

    authenticator MyAuthenticator

    routes do
      schema
      get "users", endpoint: Endpoints::Users::ListEndpoint
      post "users", endpoint: Endpoints::Users::CreateEndpoint
      get "users/:user", endpoint: Endpoints::Users::InfoEndpoint
    end
  end
end
```

## Documentation Index

- [Project Structure](project-structure.md) - How to organize your API files in a Rails app
- [The API Base Class](api-base.md) - Defining your API entry point, mounting, and configuration
- [Endpoints](endpoints.md) - Defining request handlers with arguments, fields, and actions
- [Objects](objects.md) - Defining response object types with fields and conditions
- [Argument Sets](argument-sets.md) - Defining reusable input parameter groups
- [Lookup Argument Sets](lookup-argument-sets.md) - Flexible record lookups (find by ID, slug, etc.)
- [Errors](errors.md) - Defining and raising custom API errors with exception catching
- [Enums](enums.md) - Defining fixed sets of allowed values
- [Authentication](authentication.md) - Authenticators, bearer tokens, CORS, and scopes
- [Routing](routing.md) - Route definitions, groups, HTTP methods, and path parameters
- [Scalars and Types](scalars-and-types.md) - Built-in scalar types and custom type definitions
- [Pagination](pagination.md) - Built-in pagination support for list endpoints
- [Polymorphs](polymorphs.md) - Returning different object types based on runtime conditions
- [Controllers](controllers.md) - Grouping related endpoints with shared configuration
- [Field Specs](field-specs.md) - Controlling which fields are included in responses
- [Testing](testing.md) - Testing endpoints with mock requests

## Key Concepts

Apia APIs are composed of these building blocks:

| Component | Base Class | Purpose |
|-----------|-----------|---------|
| API | `Apia::API` | Top-level entry point, defines routes and default authenticator |
| Endpoint | `Apia::Endpoint` | Individual request handler with arguments, fields, and action |
| Object | `Apia::Object` | Describes the shape of response data |
| ArgumentSet | `Apia::ArgumentSet` | Groups of input arguments for reuse |
| LookupArgumentSet | `Apia::LookupArgumentSet` | Input arguments with a resolver for finding records |
| Error | `Apia::Error` | Typed error responses with codes and HTTP statuses |
| Enum | `Apia::Enum` | Fixed set of allowed string values |
| Authenticator | `Apia::Authenticator` | Authentication and authorization logic |
| Controller | `Apia::Controller` | Groups related endpoints with shared helpers |
| Polymorph | `Apia::Polymorph` | Returns different types based on runtime matching |

## Scalar Types

Apia provides these built-in scalar types, referenced by symbol:

| Symbol | Ruby Type | Description |
|--------|-----------|-------------|
| `:string` | String | Text values |
| `:integer` | Integer | Whole numbers |
| `:boolean` | Boolean | true/false |
| `:decimal` | Float | Decimal numbers |
| `:unix_time` | Time | Unix timestamps |
| `:date` | Date | ISO 8601 dates (yyyy-mm-dd) |
| `:base64` | String | Base64-encoded binary data |
