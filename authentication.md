# Authentication

Apia provides an authenticator system for verifying requests, setting identities, configuring CORS, and validating scopes.

## Defining an Authenticator

```ruby
module MyAPI
  class MyAuthenticator < Apia::Authenticator

    name "Bearer Token Authenticator"
    description "Authenticates requests using a Bearer token in the Authorization header"

    type :bearer

    potential_error "MissingAPIToken" do
      code :missing_api_token
      description "No API token was provided in the Authorization header"
      http_status 401
    end

    potential_error "InvalidAPIToken" do
      code :invalid_api_token
      description "The API token provided was not valid"
      http_status 403
      field :details, :string, null: true
    end

    def call
      # Configure CORS
      cors.methods = %w[GET POST PUT PATCH DELETE OPTIONS]
      cors.headers = %w[Authorization Content-Type]
      cors.origin = "*"

      # Allow OPTIONS requests without authentication
      return if request.options?

      # Extract and validate the token
      given_token = request.headers["Authorization"]&.to_s&.sub(/\ABearer /, "")
      if given_token.blank?
        raise_error "MissingAPIToken"
      end

      # Look up the token and set the identity
      user = User.find_by_api_token(given_token)
      if user.nil?
        raise_error "InvalidAPIToken", details: "Token not found or expired"
      end

      request.identity = user
    end

    scope_validator do |_, _, scope|
      request.identity.scopes.include?(scope)
    end

  end
end
```

## DSL Methods

### `type(symbol)`

Sets the authenticator type. Options:
- `:bearer` - Token-based authentication via the Authorization header
- `:anonymous` - No authentication required

```ruby
type :bearer
```

### `def call` / `action(&block)`

Defines the authentication logic. Use the `call` instance method (preferred) or the `action` block.

Inside the action, you have access to:
- `request` - The current request (headers, params, etc.)
- `cors` - CORS configuration object
- `raise_error(name, **fields)` - Raise an authentication error

### `scope_validator(&block)`

Defines a block that checks whether a specific scope is granted. The block receives three arguments and should return a boolean:

```ruby
scope_validator do |_authenticator, _request, scope|
  request.identity.scopes.include?(scope)
end
```

### `potential_error(klass_or_name, &block)`

Declares errors that the authenticator may raise. Can reference existing error classes or define inline errors.

## Setting the Identity

Use `request.identity = ...` to set the authenticated identity. This can be any object - endpoints access it via `request.identity`.

```ruby
def call
  token = request.headers["Authorization"]&.sub(/\ABearer /, "")
  request.identity = User.find_by_api_token!(token)
end
```

## CORS Configuration

The `cors` object configures Cross-Origin Resource Sharing headers:

```ruby
def call
  cors.methods = %w[GET POST PUT PATCH DELETE OPTIONS]
  cors.headers = %w[Authorization Content-Type]
  cors.origin = "*"  # or "https://example.com"

  # Skip auth for preflight requests
  return if request.options?

  # ... authentication logic
end
```

## Authenticator Hierarchy

Authenticators can be set at three levels (most specific wins):

1. **Endpoint level** - Overrides controller and API authenticator
2. **Controller level** - Overrides API authenticator
3. **API level** - Default for all endpoints

```ruby
# API level (default)
class Base < Apia::API
  authenticator MyAuthenticator
end

# Endpoint level override
class CreateEndpoint < BaseEndpoint
  authenticator OAuthClientCredentialsAuthenticator
end
```

## Scopes

Scopes provide fine-grained authorization. Define available scopes in the API base class:

```ruby
class Base < Apia::API
  scopes do
    add "users.read", "Read user profiles"
    add "users.write", "Create and update users"
    add "organizations", "Access organization data"
  end
end
```

Require scopes on endpoints:

```ruby
class InfoEndpoint < BaseEndpoint
  scope "users.read"
  # or multiple
  scopes "users.read", "users.write"
end
```

The `scope_validator` block in the authenticator checks if the identity has the required scope. If not, a `ScopeNotGrantedError` (HTTP 403) is raised.

## Public (No-Auth) Authenticator

For endpoints that should be publicly accessible:

```ruby
class PublicAuthenticator < Apia::Authenticator
  name "Public Authenticator"
  description "Allows requests without any authentication."

  def call
    true
  end
end
```

Use it on specific endpoints:

```ruby
class PublicInfoEndpoint < BaseEndpoint
  authenticator PublicAuthenticator

  def call
    response.add_field :status, "ok"
  end
end
```

## Complete Examples

### Bearer Token Authenticator

```ruby
class OAuthAccessTokenAuthenticator < Apia::Authenticator
  name "OAuth Access Token Authenticator"
  description "Authenticates using an OAuth2 access token"

  type :bearer

  potential_error Errors::TokenOwnerSuspendedError

  potential_error "MissingAPIToken" do
    code :missing_api_token
    description "No API token was provided in the Authorization header"
    http_status 401
  end

  potential_error "InvalidAPIToken" do
    code :invalid_api_token
    description "The API token provided was not valid"
    http_status 403
    field :details, :string, null: true
  end

  def call
    given_token = request.headers["Authorization"]&.to_s&.sub(/\ABearer /, "")
    if given_token.blank?
      raise_error "MissingAPIToken"
    end

    user = User.find_by_api_token(given_token)
    if user.nil?
      raise_error "InvalidAPIToken"
    end

    if user.suspended?
      raise_error Errors::TokenOwnerSuspendedError
    end

    request.identity = user
  end

  scope_validator do |_, _, scope|
    request.identity.scopes.include?(scope)
  end
end
```

### Client Credentials Authenticator

```ruby
class OAuthClientCredentialsAuthenticator < Apia::Authenticator
  name "OAuth Client Credentials Authenticator"
  description "Authenticates using client credentials"

  type :bearer

  potential_error "MissingAPIToken" do
    code :missing_api_token
    http_status 401
  end

  potential_error "InvalidAPIToken" do
    code :invalid_api_token
    http_status 403
  end

  def call
    given_token = request.headers["Authorization"]&.to_s&.sub(/\ABearer /, "")
    if given_token.blank?
      raise_error "MissingAPIToken"
    end

    client = OAuthClient.find_by_token(given_token)
    if client.nil?
      raise_error "InvalidAPIToken"
    end

    request.identity = client
  end

  scope_validator do |_, _, scope|
    request.identity.scopes.exists?(name: scope)
  end
end
```
