# Endpoints

Endpoints are the core request handlers in Apia. Each endpoint defines its input arguments, output fields, authentication requirements, and the action to execute.

## Defining an Endpoint

```ruby
module MyAPI
  module Endpoints
    module Users
      class InfoEndpoint < BaseEndpoint

        name "Get user details"
        description "Returns details about the authenticated user"

        # Output fields
        field :user, Objects::User

        # Required scopes
        scopes "users.read"

        def call
          response.add_field :user, request.identity.user
        end

      end
    end
  end
end
```

## DSL Methods

### `name(string)` / `description(string)`

Set metadata for documentation and schema generation.

### `argument(name, type, **options, &block)`

Defines an input argument. Arguments are parsed from the request body (JSON) and/or URL parameters.

```ruby
# Simple scalar argument
argument :name, :string, required: true

# With description
argument :email, :string, required: true, description: "The user's email address"

# Optional with default
argument :page, :integer, default: 1

# Boolean argument
argument :active, :boolean, default: true

# Array argument
argument :tags, [:string]

# Enum argument
argument :status, Enums::StatusEnum

# Nested argument set
argument :properties, ArgumentSets::UserCreateProperties, required: true

# Lookup argument set
argument :user, ArgumentSets::UserLookup, required: true

# With custom validation
argument :age, :integer do
  validation(:must_be_positive) { |value| value.positive? }
end
```

### `field(name, type, **options, &block)`

Defines an output field in the response.

```ruby
# Simple scalar field
field :id, :string
field :name, :string
field :created_at, :unix_time

# Nullable field
field :email, :string, null: true

# Array field
field :tags, [:string]

# Object field
field :user, Objects::User

# Array of objects
field :users, [Objects::User]

# With custom backend
field :full_name, :string, backend: -> (user) { "#{user.first_name} #{user.last_name}" }

# Omit field from response entirely when nil (instead of returning null)
field :nickname, :string, null: true, skip_if_null: true

# With include directive (controls default field inclusion)
field :organization, Objects::Organization, include: "*,users[user_id,owner]"

# With description
field :status, :string, description: "The current account status"

# With pagination (see pagination.md)
field :users, [Objects::User], paginate: true
```

### `action(&block)`

Defines the endpoint action as a block (alternative to defining a `call` method):

```ruby
action do
  user = User.find_by(uuid: request.arguments[:id])
  response.add_field :user, user
end
```

### `def call`

The preferred way to define endpoint logic. Define a `call` instance method:

```ruby
def call
  user = User.find_by(uuid: request.arguments[:id])
  response.add_field :user, user
end
```

### `http_status(code)`

Sets the HTTP status code for successful responses. Defaults to 200.

```ruby
http_status 201  # For creation endpoints
```

### `authenticator(klass)`

Overrides the default authenticator for this endpoint:

```ruby
authenticator PublicAuthenticator  # No auth required
```

### `potential_error(klass, &block)`

Declares that this endpoint may raise a specific error. This is important for schema generation and documentation.

```ruby
# Reference an existing error class
potential_error Errors::ValidationError

# Define an inline error
potential_error "UserNotFound" do
  code :user_not_found
  description "The specified user could not be found"
  http_status 404
end
```

### `scope(name)` / `scopes(*names)`

Requires one or more OAuth scopes to access this endpoint:

```ruby
scope "users.read"
# or
scopes "users.read", "users.write"
```

### `response_type(type)`

Sets the response content type. Defaults to JSON. Use `Apia::Response::PLAIN` (or the string `'text/plain'`) for plain text responses:

```ruby
response_type Apia::Response::PLAIN
```

When using a plain text response type, set the response body directly:

```ruby
def call
  response.body = "Hello, world!"
end
```

**Note:** Apia also accepts JSON requests with vendor-specific content types (e.g., `application/vnd.docker.distribution.events.v2+json`). Any `application/*+json` content type will be parsed as JSON.

## Request and Response

Inside the `call` method, you have access to:

### `request`

- `request.arguments` - Parsed argument set (hash-like access with `[:key]`)
- `request.identity` - The authenticated identity (set by the authenticator)
- `request.headers` - Request headers (case-insensitive access)
- `request.ip` - Client IP address
- `request.user_agent` - Client user agent

### `response`

- `response.add_field(name, value)` - Set a response field value
- `response.add_header(name, value)` - Add a response header

## Complete Examples

### GET Endpoint (Read)

```ruby
class InfoEndpoint < BaseEndpoint
  name "Get organization info"
  description "Returns information for the requested organization"

  argument :organization, type: ArgumentSets::OrganizationLookup, required: true
  field :organization, Objects::Organization

  scopes "organizations.read"

  def call
    organization = request.arguments[:organization].resolve(
      Organization.accessible_by_user(request.identity.user)
    )
    response.add_field :organization, organization
  end
end
```

### POST Endpoint (Create)

```ruby
class CreateEndpoint < BaseEndpoint
  name "Create user"
  description "Create a new user"

  http_status 201

  authenticator OAuthClientCredentialsAuthenticator

  argument :properties, ArgumentSets::UserCreateProperties, required: true

  field :id, :string

  potential_error Errors::ValidationError

  scopes "users.create"

  def call
    attrs = request.arguments[:properties].to_hash
    user = User.create!(attrs)
    response.add_field :id, user.uuid
  end
end
```

### PATCH Endpoint (Update)

```ruby
class UpdateEndpoint < BaseEndpoint
  name "Update user details"
  description "Allows updating of basic user details"

  argument :properties, ArgumentSets::UserUpdateProperties, required: true

  field :user, Objects::User

  potential_error Errors::ValidationError

  scopes "users.update"

  def call
    props = request.arguments[:properties].to_hash
    user = request.identity.user
    user.update!(props)
    response.add_field :user, user
  end
end
```

### DELETE Endpoint

```ruby
class DeleteEndpoint < BaseEndpoint
  name "Delete SSH key"
  description "Deletes an existing SSH key"

  argument :ssh_key, ArgumentSets::SSHKeyLookup, required: true

  field :status, :boolean

  scopes "ssh_keys.delete"

  def call
    ssh_key = request.arguments[:ssh_key].resolve(request.identity.user.ssh_keys)
    ssh_key.destroy!
    response.add_field :status, true
  end
end
```

### List Endpoint

```ruby
class ListEndpoint < BaseEndpoint
  name "List organizations"
  description "Returns a list of all organizations for the user"

  argument :exclude_managed, :boolean, default: false

  field :organizations, [Objects::Organization]

  scopes "organizations.read"

  def call
    organizations = Organization.accessible_by_user(request.identity.user)
    organizations = organizations.unmanaged if request.arguments[:exclude_managed]
    response.add_field :organizations, organizations.order(:name).to_a
  end
end
```
