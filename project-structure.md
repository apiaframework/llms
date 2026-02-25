# Project Structure

Apia API files live under `app/apis/` in a Rails application. Each API is a module containing all its components organized by type.

## Directory Layout

```
app/
  apis/
    my_api/
      base.rb                          # API base class (< Apia::API)
      my_authenticator.rb              # Authenticator (< Apia::Authenticator)
      helpers.rb                       # Shared helper module (optional)
      argument_sets/
        user_lookup.rb                 # LookupArgumentSet (< Apia::LookupArgumentSet)
        user_create_properties.rb      # ArgumentSet (< Apia::ArgumentSet)
        user_update_properties.rb
        organization_lookup.rb
      endpoints/
        base_endpoint.rb               # Shared base endpoint (< Apia::Endpoint)
        users/
          info_endpoint.rb             # GET user details
          list_endpoint.rb             # GET user list
          create_endpoint.rb           # POST create user
          update_endpoint.rb           # PATCH update user
        organizations/
          info_endpoint.rb
          list_endpoint.rb
      errors/
        validation_error.rb            # Error (< Apia::Error)
        permission_denied_error.rb
        not_found_error.rb
      enums/
        color_scheme_enum.rb           # Enum (< Apia::Enum)
        status_enum.rb
      objects/
        user.rb                        # Object (< Apia::Object)
        organization.rb
```

## Naming Conventions

| Component | Class Name Pattern | File Name Pattern |
|-----------|-------------------|-------------------|
| API Base | `MyAPI::Base` | `base.rb` |
| Endpoint | `MyAPI::Endpoints::Users::InfoEndpoint` | `endpoints/users/info_endpoint.rb` |
| Object | `MyAPI::Objects::User` | `objects/user.rb` |
| ArgumentSet | `MyAPI::ArgumentSets::UserUpdateProperties` | `argument_sets/user_update_properties.rb` |
| LookupArgumentSet | `MyAPI::ArgumentSets::UserLookup` | `argument_sets/user_lookup.rb` |
| Error | `MyAPI::Errors::ValidationError` | `errors/validation_error.rb` |
| Enum | `MyAPI::Enums::StatusEnum` | `enums/status_enum.rb` |
| Authenticator | `MyAPI::MyAuthenticator` | `my_authenticator.rb` |

## Module Nesting

All components are nested under the API module:

```ruby
module MyAPI
  module Endpoints
    module Users
      class InfoEndpoint < BaseEndpoint
        # ...
      end
    end
  end
end
```

## Base Endpoint Pattern

It is conventional to define a `BaseEndpoint` class that all endpoints inherit from. This is a good place for shared private methods:

```ruby
module MyAPI
  module Endpoints
    class BaseEndpoint < Apia::Endpoint

      private

      def current_user
        request.identity.user
      end

    end
  end
end
```

## Multiple APIs

A single Rails application can serve multiple APIs. Each is a separate module with its own base class, mounted at a different path:

```ruby
# config/application.rb
config.middleware.use Apia::Rack, "CoreAPI::Base", "/api/v1", development: Rails.env.development?
config.middleware.use Apia::Rack, "UIAPI::Base", "/api/ui", development: Rails.env.development?
```
