# Testing

Apia provides built-in support for testing endpoints with mock requests.

## Testing Endpoints

Use `Endpoint.test` to execute an endpoint with a mock request:

```ruby
# Test an endpoint directly
result = MyAPI::Endpoints::Users::InfoEndpoint.test do |req|
  req.identity = mock_user
  req.arguments[:user_id] = "abc-123"
end
```

## Testing via the API

Use `API.test_endpoint` to test through the full API stack including authentication:

```ruby
result = MyAPI::Base.test_endpoint(
  MyAPI::Endpoints::Users::InfoEndpoint,
  controller: MyAPI::Controllers::UsersController
)
```

## Mock Requests

The `MockRequest` class provides a test double for HTTP requests:

- Default empty JSON body
- Default empty params
- Default IP address
- Settable identity, arguments, and headers

## Testing Patterns

### Unit Testing an Endpoint

```ruby
RSpec.describe MyAPI::Endpoints::Users::InfoEndpoint do
  it "returns user details" do
    user = create(:user)

    response = described_class.test do |req|
      req.identity = user
    end

    expect(response.status).to eq(200)
    expect(response.hash[:user][:name]).to eq(user.name)
  end
end
```

### Testing Error Cases

```ruby
it "raises an error when user not found" do
  expect {
    described_class.test do |req|
      req.identity = create(:user)
      req.arguments[:user_id] = "nonexistent"
    end
  }.to raise_error(Apia::ErrorExceptionError) do |error|
    expect(error.error_class.definition.code).to eq(:user_not_found)
  end
end
```

### Integration Testing with Rack

Since Apia runs as Rack middleware, you can also test it with standard Rack test helpers:

```ruby
RSpec.describe "Users API", type: :request do
  it "returns user info" do
    user = create(:user)
    token = create(:api_token, user: user)

    get "/api/v1/user",
      headers: { "Authorization" => "Bearer #{token.value}" }

    expect(response).to have_http_status(200)
    json = JSON.parse(response.body)
    expect(json["user"]["name"]).to eq(user.name)
  end
end
```
