# Routing

Routes map HTTP methods and URL paths to endpoints. They are defined in the API base class inside the `routes` block.

## Defining Routes

```ruby
class Base < Apia::API
  routes do
    # Auto-generated schema endpoint
    schema

    # Simple routes
    get "users", endpoint: Endpoints::Users::ListEndpoint
    post "users", endpoint: Endpoints::Users::CreateEndpoint
    get "users/:user", endpoint: Endpoints::Users::InfoEndpoint
    patch "users/:user", endpoint: Endpoints::Users::UpdateEndpoint
    delete "users/:user", endpoint: Endpoints::Users::DeleteEndpoint
  end
end
```

## HTTP Method Helpers

| Method | Usage |
|--------|-------|
| `get` | Read operations |
| `post` | Create operations |
| `patch` | Partial update operations |
| `put` | Full update / replace / synchronize operations |
| `delete` | Delete operations |

All have the same signature:

```ruby
get "path", endpoint: EndpointClass
post "path", endpoint: EndpointClass
patch "path", endpoint: EndpointClass
put "path", endpoint: EndpointClass
delete "path", endpoint: EndpointClass
```

## Path Parameters

Use `:param_name` in the path to define dynamic segments. The parameter value is extracted from the URL and passed to the endpoint's arguments:

```ruby
get "organizations/:organization", endpoint: Endpoints::Organizations::InfoEndpoint
delete "invites/:invite", endpoint: Endpoints::Invites::DeleteEndpoint
post "users/:user/emails", endpoint: Endpoints::Emails::CreateForUserEndpoint
```

When a path parameter matches a lookup argument set argument name, Apia automatically extracts the value and passes it to the argument set for resolution. For example, with route `get "organizations/:organization"` and endpoint argument `argument :organization, ArgumentSets::OrganizationLookup`, the path value is used as the lookup input.

## Route Groups

Groups organize related routes under a logical section. They appear in the generated schema and documentation.

```ruby
routes do
  schema

  group :users do
    name "Users"

    get "user", endpoint: Endpoints::Users::InfoEndpoint
    patch "user", endpoint: Endpoints::Users::UpdateEndpoint
    post "users", endpoint: Endpoints::Users::CreateEndpoint
  end

  group :organizations do
    name "Organizations"

    get "organizations", endpoint: Endpoints::Organizations::ListEndpoint
    get "organizations/:organization", endpoint: Endpoints::Organizations::InfoEndpoint
    patch "organizations/:organization", endpoint: Endpoints::Organizations::UpdateEndpoint
    post "organizations", endpoint: Endpoints::Organizations::CreateEndpoint
  end

  group :ssh_keys do
    name "SSH Keys"

    get "ssh_keys", endpoint: Endpoints::SSHKeys::ListEndpoint
    post "ssh_keys", endpoint: Endpoints::SSHKeys::CreateEndpoint
    delete "ssh_keys/:ssh_key", endpoint: Endpoints::SSHKeys::DeleteEndpoint
  end
end
```

### Group DSL Methods

- `name(string)` - Human-readable name for the group
- `description(string)` - Description of the group
- `controller(klass)` - Set a default controller for routes in this group
- `no_schema` - Exclude this group from schema generation

### Nested Groups

Groups can be nested:

```ruby
group :time do
  name "Time functions"
  description "Everything related to time elements"

  get "time/now", endpoint: Endpoints::TimeNowEndpoint

  group :formatting do
    name "Formatting"
    controller Controllers::TimeController

    get "time/formatting/format", endpoint: :format
    post "time/formatting/format", endpoint: :format
  end
end
```

## Controller Routes

When using controllers, you can reference endpoints by symbol name instead of class:

```ruby
# Direct endpoint reference
get "time/now", endpoint: Endpoints::TimeNowEndpoint

# Controller + endpoint symbol
get "time/format", controller: Controllers::TimeController, endpoint: :format

# Or set default controller on a group
group :time do
  controller Controllers::TimeController
  get "time/format", endpoint: :format
end
```

## Schema Endpoint

The `schema` call inside routes generates an auto-documentation endpoint:

```ruby
routes do
  schema  # Generates GET /schema returning the full API schema as JSON
end
```

This returns a JSON description of all routes, endpoints, arguments, fields, objects, enums, and errors.

## Same Path, Multiple Methods

The same path can handle different HTTP methods:

```ruby
get "user/authorization", endpoint: Endpoints::Users::AuthorizationEndpoint
post "user/authorization", endpoint: Endpoints::Users::AuthorizationEndpoint
```

## Complete Example

```ruby
routes do
  schema

  group :client do
    name "Client"

    get "client/users/:user", endpoint: Endpoints::Client::UserInfoEndpoint
    post "client/organizations", endpoint: Endpoints::Client::Organizations::CreateEndpoint
    get "client/organizations/:organization", endpoint: Endpoints::Client::Organizations::InfoEndpoint
    patch "client/organizations/:organization", endpoint: Endpoints::Client::Organizations::UpdateEndpoint
  end

  group :users do
    name "Users"

    get "user", endpoint: Endpoints::Users::InfoEndpoint
    patch "user", endpoint: Endpoints::Users::UpdateEndpoint
    post "users", endpoint: Endpoints::Users::CreateEndpoint
  end

  group :organizations do
    name "Organizations"

    get "organizations/:organization", endpoint: Endpoints::Organizations::InfoEndpoint
    get "organizations", endpoint: Endpoints::Organizations::ListEndpoint
    patch "organizations/:organization", endpoint: Endpoints::Organizations::UpdateEndpoint
    post "organizations", endpoint: Endpoints::Organizations::CreateEndpoint
  end

  group :invites do
    name "Invites"

    get "organizations/:organization/invites", endpoint: Endpoints::Invites::ListEndpoint
    post "organizations/:organization/invites", endpoint: Endpoints::Invites::CreateEndpoint
    delete "invites/:invite", endpoint: Endpoints::Invites::DeleteEndpoint
  end

  group :sessions do
    name "Sessions"

    put "sessions/:session", endpoint: Endpoints::Sessions::UpdateEndpoint
    post "sessions/:session/logout", endpoint: Endpoints::Sessions::LogoutEndpoint
  end
end
```
