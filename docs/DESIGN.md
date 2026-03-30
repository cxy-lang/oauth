# OAuth Client Library for Cxy - Design Document

## Overview

A simple, focused OAuth 2.0 client library for Cxy applications. This library enables Cxy apps to authenticate users via popular OAuth providers (GitHub, Google, GitLab, etc.) and access protected APIs on their behalf.

**This is a CLIENT library only** - it does NOT implement an OAuth provider/authorization server.

## Goals

1. **Simple API** - Easy-to-use interface for OAuth flows in Cxy applications
2. **Provider Support** - Pre-configured support for popular OAuth providers
3. **Secure** - PKCE support, secure token storage, and best practices by default
4. **Standards Compliant** - Implements RFC 6749 (OAuth 2.0) and RFC 7636 (PKCE)
5. **Lightweight** - Minimal dependencies, focused scope

## Supported Features

### Grant Types
- **Authorization Code Flow** - Standard server-side OAuth flow
- **Authorization Code + PKCE** - Secure flow for SPAs and mobile apps
- **Client Credentials** - Server-to-server authentication (M2M)
- **Token Refresh** - Automatic token refresh when expired

### Built-in Providers
- GitHub
- Google
- GitLab
- Microsoft/Azure AD
- Generic OAuth 2.0 provider (custom configuration)

## Architecture

```
┌─────────────────────────────────────────────────────┐
│               Cxy Application                        │
└────────────────────┬────────────────────────────────┘
                     │
                     │ uses
                     │
┌────────────────────▼────────────────────────────────┐
│            @oauth Client Library                     │
├──────────────────────────────────────────────────────┤
│                                                       │
│  OAuthClient                                         │
│  ├── Authorization Flow                              │
│  ├── Token Management                                │
│  └── HTTP Client (with auto-refresh)                │
│                                                       │
│  Provider Configurations                             │
│  ├── GitHub                                          │
│  ├── Google                                          │
│  └── Custom                                          │
│                                                       │
│  Token Storage (pluggable)                           │
│  ├── Memory                                          │
│  ├── File                                            │
│  └── Custom                                          │
│                                                       │
└────────────────────┬────────────────────────────────┘
                     │
                     │ HTTP requests
                     │
┌────────────────────▼────────────────────────────────┐
│          OAuth Provider (GitHub, Google, etc.)       │
└──────────────────────────────────────────────────────┘
```

## Module Structure

```
oauth/
├── index.cxy                          # Main package exports
├── README.md                          # Usage documentation
├── docs/
│   └── DESIGN.md                      # This file
├── src/
│   ├── client.cxy                     # Main OAuthClient class
│   ├── config.cxy                     # Client configuration
│   ├── flows/
│   │   ├── authorization_code.cxy     # Authorization code flow
│   │   ├── pkce.cxy                   # PKCE implementation
│   │   ├── client_credentials.cxy     # Client credentials flow
│   │   └── refresh.cxy                # Token refresh
│   ├── providers/
│   │   ├── base.cxy                   # Base provider interface
│   │   ├── github.cxy                 # GitHub config
│   │   ├── google.cxy                 # Google config
│   │   ├── gitlab.cxy                 # GitLab config
│   │   ├── microsoft.cxy              # Microsoft/Azure AD config
│   │   └── generic.cxy                # Generic OAuth 2.0
│   ├── token/
│   │   ├── models.cxy                 # Token data structures
│   │   ├── storage.cxy                # Token storage interface
│   │   ├── memory_storage.cxy         # In-memory storage
│   │   └── file_storage.cxy           # File-based storage
│   ├── http/
│   │   ├── client.cxy                 # HTTP client wrapper
│   │   └── interceptors.cxy           # Auth interceptors
│   └── utils/
│       ├── crypto.cxy                 # Crypto helpers (PKCE, state)
│       ├── validation.cxy             # Input validation
│       └── errors.cxy                 # Error types
└── test/
    ├── client_test.cxy
    ├── flows_test.cxy
    └── providers_test.cxy
```

## Core API Design

### OAuthClient

The main entry point for OAuth operations:

```cxy
class OAuthClient {
  config: OAuthConfig
  tokenStorage: TokenStorage
  httpClient: http.Client
  
  pub func new(config: OAuthConfig) -> OAuthClient
  pub func withStorage(storage: TokenStorage) -> OAuthClient
  
  // Authorization flow
  pub func getAuthorizationUrl(state: ?string = null) -> string
  pub func handleCallback(callbackUrl: string) -> Result[TokenResponse, OAuthError]
  
  // Token management
  pub func refreshToken(refreshToken: string) -> Result[TokenResponse, OAuthError]
  pub func revokeToken(token: string) -> Result[void, OAuthError]
  
  // Authenticated requests
  pub func get(url: string, token: string) -> Result[http.Response, OAuthError]
  pub func post(url: string, token: string, body: ?string) -> Result[http.Response, OAuthError]
  
  // Helper for automatic token refresh
  pub func createAuthedClient(accessToken: string, refreshToken: ?string = null) -> AuthedHttpClient
}
```

### OAuthConfig

Configuration for OAuth client:

```cxy
class OAuthConfig {
  // Required
  clientId: string
  clientSecret: ?string         // Optional for public clients
  redirectUri: string
  
  // Provider endpoints
  authorizationEndpoint: string
  tokenEndpoint: string
  revocationEndpoint: ?string
  
  // Optional settings
  scopes: []string              // Default: []
  usePKCE: bool                 // Default: true
  state: ?string                // Auto-generated if not provided
  
  // PKCE settings
  pkceMethod: string            // "S256" or "plain", default: "S256"
  
  // Additional params
  additionalAuthParams: ?HashMap[string, string]
  additionalTokenParams: ?HashMap[string, string]
}
```

### Provider Configurations

Pre-configured providers for convenience:

```cxy
module providers

// GitHub
pub func GitHub(clientId: string, clientSecret: string, redirectUri: string, scopes: []string = ["user"]) -> OAuthConfig {
  return OAuthConfig {
    clientId: clientId,
    clientSecret: clientSecret,
    redirectUri: redirectUri,
    authorizationEndpoint: "https://github.com/login/oauth/authorize",
    tokenEndpoint: "https://github.com/login/oauth/access_token",
    scopes: scopes,
    usePKCE: false,  // GitHub doesn't require PKCE
  }
}

// Google
pub func Google(clientId: string, clientSecret: string, redirectUri: string, scopes: []string = ["openid", "profile", "email"]) -> OAuthConfig {
  return OAuthConfig {
    clientId: clientId,
    clientSecret: clientSecret,
    redirectUri: redirectUri,
    authorizationEndpoint: "https://accounts.google.com/o/oauth2/v2/auth",
    tokenEndpoint: "https://oauth2.googleapis.com/token",
    revocationEndpoint: "https://oauth2.googleapis.com/revoke",
    scopes: scopes,
    usePKCE: true,
  }
}

// GitLab
pub func GitLab(clientId: string, clientSecret: string, redirectUri: string, scopes: []string = ["read_user"]) -> OAuthConfig {
  return OAuthConfig {
    clientId: clientId,
    clientSecret: clientSecret,
    redirectUri: redirectUri,
    authorizationEndpoint: "https://gitlab.com/oauth/authorize",
    tokenEndpoint: "https://gitlab.com/oauth/token",
    scopes: scopes,
    usePKCE: true,
  }
}

// Microsoft/Azure AD
pub func Microsoft(clientId: string, clientSecret: string, redirectUri: string, tenant: string = "common", scopes: []string = ["openid", "profile", "email"]) -> OAuthConfig {
  return OAuthConfig {
    clientId: clientId,
    clientSecret: clientSecret,
    redirectUri: redirectUri,
    authorizationEndpoint: f"https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize",
    tokenEndpoint: f"https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token",
    scopes: scopes,
    usePKCE: true,
  }
}

// Generic OAuth 2.0 provider
pub func Generic(config: OAuthConfig) -> OAuthConfig {
  return config
}
```

### Token Models

```cxy
class TokenResponse {
  accessToken: string
  tokenType: string           // Usually "Bearer"
  expiresIn: ?i64            // Seconds until expiration
  refreshToken: ?string
  scope: ?string
  
  // Computed
  expiresAt: ?i64            // Unix timestamp
  
  pub func isExpired() -> bool
  pub func expiresInSeconds() -> ?i64
}

class TokenStorage {
  pub func save(key: string, token: TokenResponse) -> Result[void, Error]
  pub func load(key: string) -> Result[?TokenResponse, Error]
  pub func delete(key: string) -> Result[void, Error]
  pub func clear() -> Result[void, Error]
}
```

## Usage Examples

### 1. Basic Authorization Code Flow (Server-Side)

```cxy
import { OAuthClient } from "@oauth"
import { GitHub } from "@oauth/providers"
import { http } from "std"

func main() {
  let server = http.Server.new()
  
  let oauth = OAuthClient.new(
    GitHub(
      clientId: env("GITHUB_CLIENT_ID"),
      clientSecret: env("GITHUB_CLIENT_SECRET"),
      redirectUri: "http://localhost:3000/callback",
      scopes: ["user", "repo"]
    )
  )
  
  // Login route - redirect to GitHub
  server.get("/login", func(req, res) {
    let authUrl = oauth.getAuthorizationUrl()
    res.redirect(authUrl)
  })
  
  // Callback route - handle GitHub redirect
  server.get("/callback", func(req, res) {
    let callbackUrl = f"http://localhost:3000{req.url}"
    
    match oauth.handleCallback(callbackUrl) {
      Ok(tokens) => {
        // Save tokens (e.g., in session)
        req.session.set("access_token", tokens.accessToken)
        req.session.set("refresh_token", tokens.refreshToken)
        
        res.redirect("/dashboard")
      }
      Err(e) => {
        res.status(400).send(f"OAuth error: {e}")
      }
    }
  })
  
  // Protected route - use access token
  server.get("/dashboard", func(req, res) {
    let token = req.session.get("access_token")?
    
    // Make authenticated request to GitHub API
    match oauth.get("https://api.github.com/user", token) {
      Ok(response) => {
        res.send(f"User data: {response.body}")
      }
      Err(e) => {
        res.status(500).send(f"Error: {e}")
      }
    }
  })
  
  server.listen(3000)
  println("Server running on http://localhost:3000")
}
```

### 2. Authorization Code Flow with PKCE (SPA/Mobile)

```cxy
import { OAuthClient } from "@oauth"
import { Google } from "@oauth/providers"

func main() {
  let oauth = OAuthClient.new(
    Google(
      clientId: env("GOOGLE_CLIENT_ID"),
      clientSecret: null,  // Public client - no secret
      redirectUri: "http://localhost:3000/callback",
      scopes: ["openid", "profile", "email"]
    )
  )
  
  // PKCE is enabled by default for Google
  let authUrl = oauth.getAuthorizationUrl()
  println(f"Visit: {authUrl}")
  
  // After user authorizes and is redirected back...
  let callbackUrl = readline("Enter callback URL: ")
  
  match oauth.handleCallback(callbackUrl) {
    Ok(tokens) => {
      println(f"Access Token: {tokens.accessToken}")
      println(f"Expires in: {tokens.expiresIn} seconds")
      
      // Use the token
      match oauth.get("https://www.googleapis.com/oauth2/v1/userinfo", tokens.accessToken) {
        Ok(response) => {
          println(f"User info: {response.body}")
        }
        Err(e) => {
          println(f"Error: {e}")
        }
      }
    }
    Err(e) => {
      println(f"OAuth error: {e}")
    }
  }
}
```

### 3. Client Credentials Flow (M2M)

```cxy
import { OAuthClient, ClientCredentialsFlow } from "@oauth"
import { Generic } from "@oauth/providers"

func main() {
  let oauth = OAuthClient.new(
    Generic({
      clientId: env("CLIENT_ID"),
      clientSecret: env("CLIENT_SECRET"),
      redirectUri: "",  // Not used for client credentials
      tokenEndpoint: "https://auth.example.com/oauth/token",
      authorizationEndpoint: "",  // Not used
      scopes: ["api:read", "api:write"],
    })
  )
  
  let flow = ClientCredentialsFlow.new(&oauth)
  
  match flow.getToken() {
    Ok(tokens) => {
      println(f"Access Token: {tokens.accessToken}")
      
      // Make API request
      match oauth.get("https://api.example.com/resource", tokens.accessToken) {
        Ok(response) => {
          println(f"Response: {response.body}")
        }
        Err(e) => {
          println(f"Error: {e}")
        }
      }
    }
    Err(e) => {
      println(f"OAuth error: {e}")
    }
  }
}
```

### 4. Automatic Token Refresh

```cxy
import { OAuthClient, AuthedHttpClient } from "@oauth"
import { GitHub } from "@oauth/providers"

func main() {
  let oauth = OAuthClient.new(
    GitHub(
      clientId: env("GITHUB_CLIENT_ID"),
      clientSecret: env("GITHUB_CLIENT_SECRET"),
      redirectUri: "http://localhost:3000/callback"
    )
  )
  
  // Assume we already have tokens from previous auth
  let accessToken = "existing_access_token"
  let refreshToken = "existing_refresh_token"
  
  // Create HTTP client with automatic token refresh
  let client = oauth.createAuthedClient(accessToken, refreshToken)
  
  // Make requests - tokens are refreshed automatically when expired
  match client.get("https://api.github.com/user") {
    Ok(response) => {
      println(f"User: {response.body}")
      
      // Get updated token after potential refresh
      let currentToken = client.getCurrentToken()
      println(f"Current token: {currentToken}")
    }
    Err(e) => {
      println(f"Error: {e}")
    }
  }
}
```

### 5. Custom Token Storage

```cxy
import { OAuthClient, TokenStorage } from "@oauth"
import { FileStorage } from "@oauth/token"
import { GitHub } from "@oauth/providers"

func main() {
  // Use file-based token storage
  let storage = FileStorage.new("./.tokens")
  
  let oauth = OAuthClient.new(
    GitHub(
      clientId: env("GITHUB_CLIENT_ID"),
      clientSecret: env("GITHUB_CLIENT_SECRET"),
      redirectUri: "http://localhost:3000/callback"
    )
  ).withStorage(storage)
  
  // Tokens are automatically saved/loaded from storage
  let authUrl = oauth.getAuthorizationUrl()
  // ... rest of flow
}
```

## Security Considerations

### 1. PKCE (Proof Key for Code Exchange)

- **Enabled by default** for all flows except providers that don't support it (e.g., GitHub)
- Uses SHA-256 challenge method by default
- Protects against authorization code interception attacks

### 2. State Parameter

- **Auto-generated** if not provided
- Validated on callback to prevent CSRF attacks
- Stored securely during the flow

### 3. Redirect URI Validation

- Must exactly match the registered redirect URI
- No wildcard matching for security

### 4. Token Storage

- Tokens should be stored securely (encrypted at rest)
- Never log or expose tokens in error messages
- Use HttpOnly cookies for web applications when possible

### 5. HTTPS Only

- All OAuth flows should use HTTPS in production
- Development mode can use HTTP for localhost only

### 6. Client Secret Protection

- Never expose client secrets in client-side code
- Use public clients (no secret) for SPAs and mobile apps
- Store secrets in environment variables, not in code

## Error Handling

```cxy
enum OAuthError {
  // OAuth protocol errors
  InvalidRequest(string)
  InvalidClient(string)
  InvalidGrant(string)
  UnauthorizedClient(string)
  UnsupportedGrantType(string)
  InvalidScope(string)
  
  // Client errors
  InvalidState(string)
  InvalidCallback(string)
  TokenExpired
  NetworkError(string)
  
  // Generic
  Unknown(string)
}

// All OAuth operations return Result[T, OAuthError]
```

## Testing

### Unit Tests

- PKCE code verifier/challenge generation
- State generation and validation
- Token expiration checks
- URL building and parsing

### Integration Tests

- Mock OAuth provider responses
- Complete authorization flow simulation
- Token refresh flow
- Error handling scenarios

### Provider Tests

- Validate provider configurations
- Test against OAuth provider test endpoints (where available)

## Dependencies

- `std::http` - HTTP client
- `std::json` - JSON parsing
- `std::crypto` - SHA-256 for PKCE, random generation
- `std::base64` - Base64 encoding for PKCE

## Future Enhancements

1. **Device Code Flow** - For CLI tools and headless apps
2. **OpenID Connect** - ID token validation, UserInfo endpoint
3. **Token Introspection** - RFC 7662 support
4. **Dynamic Client Registration** - RFC 7591 support
5. **More Providers** - Discord, Twitter, Facebook, etc.
6. **Token Encryption** - Encrypt tokens at rest in storage
7. **Retry Logic** - Automatic retries with exponential backoff
8. **Rate Limiting** - Built-in rate limit handling

## Implementation Phases

### Phase 1: Core Functionality (MVP)
- [x] Design document
- [ ] Basic OAuthClient implementation
- [ ] Authorization Code flow
- [ ] PKCE support
- [ ] Token refresh
- [ ] GitHub provider
- [ ] In-memory token storage
- [ ] Basic tests

### Phase 2: Additional Features
- [ ] Google, GitLab, Microsoft providers
- [ ] File-based token storage
- [ ] Client credentials flow
- [ ] Authenticated HTTP client with auto-refresh
- [ ] Comprehensive error handling
- [ ] Integration tests

### Phase 3: Polish
- [ ] Documentation and examples
- [ ] More providers
- [ ] Performance optimization
- [ ] Security audit
- [ ] Production testing