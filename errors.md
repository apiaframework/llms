# Errors

Apia provides a structured error system with typed error responses, HTTP status codes, and automatic exception catching.

## Defining an Error

```ruby
module MyAPI
  module Errors
    class ValidationError < Apia::Error

      name "Validation error"
      description "An error occurred with the input provided"

      code :validation_error
      http_status 422

      field :details, [:string]

    end
  end
end
```

## DSL Methods

### `code(symbol)`

Sets the error code returned in the response. Must be a symbol.

```ruby
code :validation_error
code :permission_denied
code :user_not_found
```

### `http_status(integer)`

Sets the HTTP status code for this error response.

```ruby
http_status 404  # Not found
http_status 403  # Forbidden
http_status 422  # Unprocessable entity
http_status 401  # Unauthorized
```

### `description(string)`

A human-readable description of when this error occurs.

### `field(name, type, **options)`

Adds extra data fields to the error response. Follows the same syntax as object fields.

```ruby
field :details, [:string]
field :details, :string, null: true do
  description "Additional information regarding the reason"
end
field :given_token, :string
```

### `catch_exception(klass, &block)`

Maps a Ruby exception class to this error. When the exception is raised inside an endpoint action, it is automatically caught and converted to this API error.

```ruby
catch_exception ActiveRecord::RecordInvalid do |fields, exception|
  fields[:details] = exception.record.errors.full_messages
end

catch_exception ServiceErrors::ValidationError do |fields, exception|
  fields[:details] = exception.errors
end
```

The block receives:
- `fields` - A hash to populate with error field values
- `exception` - The caught exception instance

## Raising Errors

### In Endpoints

Use `raise_error` inside endpoint actions:

```ruby
def call
  user = User.find_by(uuid: request.arguments[:id])
  if user.nil?
    raise_error Errors::UserNotFoundError
  end

  unless user.active?
    raise_error Errors::PermissionDeniedError, details: "User account is suspended"
  end
end
```

### By Error Class

```ruby
raise_error Errors::ValidationError, details: ["Name is required"]
raise_error Errors::PermissionDeniedError
```

### By Inline Error Name

When errors are defined inline (in an authenticator, endpoint, or lookup), reference them by their string name:

```ruby
raise_error "UserNotFound"
raise_error "InvalidAPIToken", details: "Token has expired"
```

### By Full Path

You can also use the full error ID path:

```ruby
raise_error "CoreAPI/MainAuthenticator/InvalidToken", given_token: token
```

## Declaring Potential Errors

Endpoints and authenticators must declare which errors they may raise using `potential_error`:

```ruby
class CreateEndpoint < BaseEndpoint
  # Reference an existing error class
  potential_error Errors::ValidationError
  potential_error Errors::PermissionDeniedError

  # Define an inline error
  potential_error "DuplicateEmail" do
    code :duplicate_email
    description "An account with this email already exists"
    http_status 409
  end

  def call
    raise_error Errors::ValidationError, details: ["Invalid input"]
    # or
    raise_error "DuplicateEmail"
  end
end
```

## Error Response Format

When an error is raised, the response body has this structure:

```json
{
  "error": {
    "code": "validation_error",
    "description": "An error occurred with the input provided",
    "detail": {
      "details": ["Name is required", "Email is invalid"]
    }
  }
}
```

## Built-in Errors

Apia provides these built-in error types that are raised automatically:

### `MissingArgumentError`

Raised when a required argument is not provided.

```json
{
  "error": {
    "code": "missing_required_argument",
    "description": "...",
    "detail": { "path": ["properties", "name"] }
  }
}
```

### `InvalidArgumentError`

Raised when an argument fails validation, parsing, or type checking.

```json
{
  "error": {
    "code": "invalid_argument",
    "description": "...",
    "detail": { "path": ["age"], "issue": "validation_error" }
  }
}
```

### `ScopeNotGrantedError`

Raised when the authenticated identity does not have the required scope. Returns HTTP 403.

## Complete Examples

### Simple Error (No Fields)

```ruby
class TokenOwnerSuspendedError < Apia::Error
  code :token_owner_suspended
  http_status 403
  description "The owner of this token has been suspended"
end
```

### Error with Fields

```ruby
class PermissionDeniedError < Apia::Error
  code :permission_denied
  http_status 403
  description "The authenticated identity is not permitted to perform this action"
  field :details, :string, null: true do
    description "Additional information regarding the reason why permission was denied"
  end
end
```

### Error with Exception Catching

```ruby
class ValidationError < Apia::Error
  name "Validation error"
  description "An error occurred with the input provided"

  code :validation_error
  http_status 422

  field :details, [:string]

  catch_exception ActiveRecord::RecordInvalid do |fields, exception|
    fields[:details] = exception.record.errors.full_messages
  end

  catch_exception ServiceErrors::ValidationError do |fields, exception|
    fields[:details] = exception.errors
  end
end
```
