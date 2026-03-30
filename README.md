# @oauth - OAuth 2.0 Client Library for Cxy

A simple, focused OAuth 2.0 client library for Cxy applications. Enables authentication via popular OAuth providers like GitHub, Google, GitLab, and more.

## Features

- **Authorization Code Flow** - Standard OAuth 2.0 flow for web applications
- **PKCE Support** - Proof Key for Code Exchange for enhanced security
- **Token Refresh** - Automatic token refresh support
- **User Info Fetching** - Get user profiles from OAuth providers
- **Built-in Providers** - Pre-configured settings for GitHub (more coming soon)
- **Type-Safe** - Full Cxy type safety with JSON parsing via `@json` annotations
- **Simple API** - Easy-to-use client interface

## Installation

Add to your `Cxyfile.yaml`:

```yaml
dependencies:
  - name: oauth
    git: https://github.com/user/oauth.git
    version: "0.1.0"
```

Then run:

```bash
cxy package install
```

## Quick Start

### GitHub OAuth Example

```cxy
import { OAuthClient } from "@oauth"
import { GitHub, getGitHubUser } from "@oauth/providers/github"
import { HashMap } from "stdlib/hash.cxy"

func main(): !void {
    // Create OAuth client with GitHub configuration
    var config = GitHub(
        "your-client-id".S,
        "your-client-secret".S,
        "http://localhost:8080/callback".S
    )
    
    var oauth = OAuthClient(&&config)
    
    // Step 1: Get authorization URL and redirect user
    var authUrl = oauth.getAuthorizationUrl()
    println(f"Visit: {authUrl}")
    
    // Step 2: After user authorizes, handle the callback
    // (params come from the callback URL query parameters)
    var params = HashMap[String, String]()
    params.["code".S] = "authorization-code-from-callback".S
    params.["state".S] = "state-from-callback".S
    
    var tokens = oauth.handleCallback(&params)
    
    // Step 3: Fetch user info
    var user = getGitHubUser(tokens.accessToken)
    println(f"Welcome, {user.login}!")
}
```

### Web Server Integration

```cxy
import { Server, Config, Request, Response, Status } from "stdlib/http.cxy"
import { Address } from "stdlib/net.cxy"
import { HashMap } from "stdlib/hash.cxy"
import { OAuthClient } from "@oauth"
import { GitHub, getGitHubUser } from "@oauth/providers/github"

type Endpoint = Server

var oauth: OAuthClient = null

func main(): !void {
    var config = GitHub(
        String(getenv("GITHUB_CLIENT_ID")),
        String(getenv("GITHUB_CLIENT_SECRET")),
        "http://localhost:8080/callback".S
    )
    oauth = OAuthClient(&&config)
    
    var server = Endpoint(Config{address: Address("0.0.0.0", 8080)})
    
    // Login route - redirect to GitHub
    server("GET /login", (@unused req: &const Request, resp: &Response) => {
        var authUrl = oauth.getAuthorizationUrl() catch {
            resp.end(Status.InternalError)
            return
        }
        resp.redirect(authUrl.__str())
    })
    
    // Callback route - exchange code for tokens
    server("GET /callback", (req: &const Request, resp: &Response) => {
        var params = HashMap[String, String]()
        
        var code = req.qparam("code".s)
        if !!code && !code&.empty() {
            params.["code".S] = String(__copy!(*code))
        }
        
        var state = req.qparam("state".s)
        if !!state && !state&.empty() {
            params.["state".S] = String(__copy!(*state))
        }
        
        var tokens = oauth.handleCallback(&params) catch {
            resp.body() << "Login failed: " << ex!.what()
            return
        }
        
        // Fetch user info
        var user = getGitHubUser(tokens.accessToken) catch {
            resp.body() << "Failed to get user info"
            return
        }
        
        resp.body() << "Welcome, " << user.login << "!"
        oauth.clearState()
    }).setAttrs({ parseQueryParams: true })
    
    server.start()
}
```

## API Reference

### OAuthClient

The main client class for OAuth operations.

```cxy
pub class OAuthClient {
    // Create a new OAuth client with the given configuration
    func `init`(config: OAuthConfig)
    
    // Get the authorization URL to redirect the user to
    // Generates state and PKCE parameters automatically
    func getAuthorizationUrl(customState: String? = null): !String
    
    // Handle the OAuth callback and exchange code for tokens
    // params should contain: code, state, and optionally error/error_description
    func handleCallback(params: &HashMap[String, String]): !TokenResponse
    
    // Exchange an authorization code for tokens (called by handleCallback)
    func exchangeCode(code: String): !TokenResponse
    
    // Refresh an access token using a refresh token
    func refreshToken(token: String): !TokenResponse
    
    // Get the current configuration
    func config(): &OAuthConfig
    
    // Check if there's a pending authorization
    func hasPendingAuth(): bool
    
    // Clear the pending authorization state
    func clearState()
}
```

### OAuthConfig

Configuration for the OAuth client.

```cxy
pub struct OAuthConfig {
    clientId: String                    // OAuth client ID
    clientSecret: String?               // Client secret (optional for public clients)
    redirectUri: String                 // Callback URL
    authorizationEndpoint: String       // Provider's authorization URL (full URL)
    providerUrl: String                 // Provider's base URL (e.g., "https://github.com")
    tokenPath: String                   // Token endpoint path (e.g., "/login/oauth/access_token")
    revocationPath: String?             // Token revocation path (optional)
    scopes: Vector[String]              // Requested scopes
    usePKCE: bool                       // Enable PKCE (default: true)
    pkceMethod: String                  // PKCE method: "S256" or "plain"
}
```

### OAuthConfigBuilder

Builder pattern for creating custom OAuth configurations.

```cxy
import { OAuthConfigBuilder } from "@oauth"

var builder = OAuthConfigBuilder("client-id".S, "http://localhost:8080/callback".S)
builder.authorizationEndpoint("https://provider.com/oauth/authorize".S)
builder.providerUrl("https://provider.com".S)
builder.tokenPath("/oauth/token".S)
builder.clientSecret("client-secret".S)
builder.addScope("read".S)
builder.addScope("write".S)
builder.usePKCE(true)
builder.pkceMethod("S256".S)

var config = builder.build()
var client = OAuthClient(&&config)
```

**Builder Methods:**
- `authorizationEndpoint(url: String): &This` - Set authorization endpoint
- `providerUrl(url: String): &This` - Set provider base URL
- `tokenPath(path: String): &This` - Set token endpoint path
- `revocationPath(path: String): &This` - Set revocation endpoint path
- `clientSecret(secret: String): &This` - Set client secret
- `addScope(scope: String): &This` - Add an OAuth scope
- `usePKCE(enabled: bool): &This` - Enable/disable PKCE
- `pkceMethod(method: String): &This` - Set PKCE method ("S256" or "plain")
- `build(): OAuthConfig` - Build the final configuration

### TokenResponse

Token response from the OAuth provider.

```cxy
pub struct TokenResponse {
    accessToken: String                 // The access token
    tokenType: String                   // Token type (usually "Bearer")
    expiresIn: i64?                     // Seconds until expiration
    refreshToken: String?               // Refresh token (if provided)
    scope: String?                      // Granted scopes
    
    func isExpired(): bool              // Check if token is expired
    func expiresInSeconds(): i64?       // Seconds until expiration
    func expiresAt(): i64?              // Unix timestamp of expiration
}
```

### Providers

#### GitHub

```cxy
import { GitHub, GitHubWithScopes, GitHubUser, getGitHubUser } from "@oauth/providers/github"

// Basic GitHub config (includes "user" scope by default)
var config = GitHub(clientId, clientSecret, redirectUri)

// GitHub config with custom scopes
var config = GitHubWithScopes(clientId, clientSecret, redirectUri, ["user", "repo"])

// Fetch user info after authentication
var user = getGitHubUser(tokens.accessToken)
println(f"User: {user.login}, Email: {user.email}")
```

**GitHubUser struct:**
```cxy
pub struct GitHubUser {
    id: i64                    // GitHub user ID
    login: String              // Username
    email: String?             // Email (may be null if private)
    name: String?              // Display name
    avatarUrl: String?         // Profile picture URL
    htmlUrl: String?           // GitHub profile URL
    bio: String?               // User bio
    company: String?           // Company
    location: String?          // Location
    publicRepos: i64           // Number of public repositories
    followers: i64             // Follower count
    following: i64             // Following count
}
```

**Available GitHub Scopes:**
- `user` - Read user profile
- `user:email` - Read user email addresses
- `repo` - Full repository access
- `public_repo` - Public repository access only
- `read:org` - Read organization membership

### Token Storage

The `TokenStorage` interface allows you to persist tokens between requests.

```cxy
pub class TokenStorage {
    virtual func save(key: String, token: TokenResponse): void
    virtual func load(key: String): TokenResponse?
    virtual func remove(key: String): void
    virtual func clear(): void
}
```

#### MemoryStorage

Built-in in-memory storage implementation (tokens are lost when program exits):

```cxy
import { MemoryStorage } from "@oauth/token"

// Create in-memory storage
var storage = MemoryStorage()

// Save a token
storage.save("user-123".S, tokens)

// Load a token
var loaded = storage.load("user-123".S)
if !!loaded {
    println(f"Token: {(*loaded).accessToken}")
}

// Remove a token
storage.remove("user-123".S)

// Clear all tokens
storage.clear()

// Check if token exists
if storage.has("user-123".S) {
    println("Token exists")
}

// Get number of stored tokens
var count = storage.size()
```

**Note:** For production use, implement a persistent storage backend (database, file system, etc.) by extending the `TokenStorage` class.

## Error Handling

All OAuth operations can throw exceptions:

```cxy
import {
    InvalidStateError,              // State mismatch (CSRF protection)
    MissingAuthorizationCodeError,  // No code in callback
    MissingStateError,              // No state in callback
    TokenExpiredError,              // Token has expired
    NetworkError,                   // HTTP request failed
    InvalidResponseError,           // Invalid response from provider
    InvalidRequestError,            // OAuth invalid_request error
    InvalidGrantError,              // OAuth invalid_grant error
    AccessDeniedError,              // User denied access
} from "@oauth"

var tokens = oauth.handleCallback(&params) catch {
    println(f"OAuth error: {ex!.what()}")
}
```

## Examples

See the `example/` directory for complete examples:

- `example/github_oauth.cxy` - GitHub OAuth with web server and user info display

## Security Considerations

1. **Always use HTTPS** in production
2. **Store client secrets securely** - use environment variables
3. **Validate state parameter** - protects against CSRF attacks
4. **Use PKCE** when possible - enabled by default
5. **Store tokens securely** - use HttpOnly cookies or encrypted storage

## License

MIT