---
title: "OAuth2 Explained Like You're Building It (Not Just Using It)"
date: 2026-08-24
categories: [Spring Boot, Security]
tags: [oauth2, security, spring-boot, jwt, authentication]
description: "Stop memorizing abstract flow diagrams. This post walks through building OAuth2 from the wire up — actual HTTP requests, token anatomy, PKCE, refresh tokens, and JWT validation in Spring Boot."
mermaid: true
---
## Why Most OAuth2 Explanations Fail

Every OAuth2 tutorial I've read starts with the same abstract diagram — boxes with arrows, "Resource Owner," "Authorization Server," and "Client." You nod along, then can't debug a 401 error in production.

After 17 years of building authentication systems across banking, fintech, and e-commerce in Singapore, I've learned that the only way to truly understand OAuth2 is to see what's happening on the wire. What bytes are actually being exchanged. What each token contains. Why things break.

Let's build it from scratch.

## The Core Problem OAuth2 Solves

Before OAuth2, if App A wanted to access your data on Service B, you'd give App A your Service B password. That's insane — but that's literally how early Twitter integrations worked.

OAuth2 solves this with **delegated authorization**. You grant App A a limited, revocable token to access specific resources on Service B, without ever sharing your password.

The key insight: **authentication** (proving who you are) and **authorization** (proving what you're allowed to do) are separate concerns. OAuth2 handles authorization. OpenID Connect (OIDC) layers authentication on top.

## The Authorization Code Flow — On the Wire

Let's trace an actual Authorization Code flow. This is what happens when a user clicks "Login with Google" on your app.

### Step 1: Your App Redirects to the Authorization Server

```
GET https://auth.example.com/oauth2/authorize?
    response_type=code&
    client_id=my-app-id&
    redirect_uri=https://myapp.com/callback&
    scope=openid profile email&
    state=xyzrandom123
HTTP/1.1
Host: auth.example.com
```

**response_type=code** — We want an authorization code (not a token directly)

**client_id** — Your app's registered identifier

**redirect_uri** — Where to send the user back after authorization

**scope** — What permissions you're requesting

**state** — CSRF protection. You generate this randomly and verify it on callback

### Step 2: User Authenticates and Consents

The authorization server shows a login page. User enters credentials. If valid, the server shows a consent screen: "My App wants to access your email and profile. Allow?"

This happens entirely on the authorization server's domain. Your app never sees the password.

### Step 3: Authorization Server Redirects Back with a Code

```
HTTP/1.1 302 Found
Location: https://myapp.com/callback?
    code=SplxlOBeZQQYbYS6WxSbIA&
    state=xyzrandom123
```

The `code` is short-lived (typically 30–60 seconds), single-use, and bound to the client_id and redirect_uri. It's useless without your client secret.

### Step 4: Your Backend Exchanges the Code for Tokens

```
POST https://auth.example.com/oauth2/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
client_id=my-app-id&
client_secret=my-app-secret&
redirect_uri=https://myapp.com/callback
```

This is a **back-channel** request — server-to-server. The client secret never touches the browser.

### Step 5: Authorization Server Responds with Tokens

```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "token_type": "Bearer",
    "expires_in": 3600,
    "refresh_token": "dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4",
    "id_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9...",
    "scope": "openid profile email"
}
```

You now have three tokens. Let's decode what each one actually is.

## Anatomy of an Access Token (JWT)

Most modern implementations use JWTs as access tokens. Let's decode one:

### Header

```json
{
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-id-2024-01"
}
```

**alg** — Signing algorithm (RS256 = RSA with SHA-256)

**kid** — Key ID. Used to look up the public key for verification

### Payload (Claims)

```json
{
    "iss": "https://auth.example.com",
    "sub": "user-12345",
    "aud": "my-app-id",
    "exp": 1700000000,
    "iat": 1699996400,
    "scope": "openid profile email",
    "client_id": "my-app-id",
    "roles": ["ROLE_USER", "ROLE_PREMIUM"]
}
```

**iss** (issuer) — Who created this token

**sub** (subject) — Who this token represents

**aud** (audience) — Who this token is intended for

**exp** (expiration) — Unix timestamp. After this, the token is invalid

**iat** (issued at) — When the token was created

**scope** — Granted permissions

### Signature

The signature is `RS256(base64(header) + "." + base64(payload), private_key)`. To verify, you use the authorization server's public key (fetched from its JWKS endpoint).

## Why Refresh Tokens Exist

Access tokens are intentionally short-lived (15 minutes to 1 hour). Why?

- If an access token leaks, the damage window is small
- You can't revoke a JWT-based access token (it's stateless)
- Short-lived tokens force regular checks with the auth server

But you don't want users re-logging-in every hour. Enter refresh tokens.

A refresh token is:
- Long-lived (days to months)
- Stored securely server-side
- Used only in back-channel requests
- Revocable (the auth server tracks them in a database)
- Rotated on each use (old refresh token invalidated, new one issued)

### The Refresh Flow

```
POST https://auth.example.com/oauth2/token HTTP/1.1
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&
refresh_token=dGhpcyBpcyBhIHJlZnJlc2ggdG9rZW4&
client_id=my-app-id&
client_secret=my-app-secret
```

Response: new access token + new refresh token. The old refresh token is immediately invalidated.

If someone steals a refresh token and uses it, the legitimate user's next refresh will fail (because the token was already rotated). This triggers a security alert and full session revocation.

## PKCE: Securing Public Clients

Mobile apps and SPAs can't keep a client secret. Anyone can decompile an APK or inspect JavaScript. PKCE (Proof Key for Code Exchange, pronounced "pixy") solves this.

### How PKCE Works

**Step 1** — Client generates a random `code_verifier` (43–128 characters)

```
code_verifier = "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
```

**Step 2** — Client computes `code_challenge = BASE64URL(SHA256(code_verifier))`

```
code_challenge = "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"
```

**Step 3** — Authorization request includes the challenge

```
GET https://auth.example.com/oauth2/authorize?
    response_type=code&
    client_id=mobile-app&
    redirect_uri=myapp://callback&
    scope=openid profile&
    code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM&
    code_challenge_method=S256&
    state=abc123
```

**Step 4** — Token exchange includes the verifier

```
POST https://auth.example.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code&
code=SplxlOBeZQQYbYS6WxSbIA&
redirect_uri=myapp://callback&
client_id=mobile-app&
code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

The auth server hashes the verifier and compares it to the stored challenge. Even if an attacker intercepts the authorization code, they can't exchange it without the original verifier.

**Important** — PKCE is now recommended for ALL OAuth2 clients, not just public ones. It protects against authorization code interception attacks.

## Token Introspection

For opaque (non-JWT) tokens, the resource server needs to validate tokens by calling the authorization server:

```
POST https://auth.example.com/oauth2/introspect HTTP/1.1
Content-Type: application/x-www-form-urlencoded
Authorization: Basic bXktYXBwLWlkOm15LWFwcC1zZWNyZXQ=

token=access-token-value-here
```

Response:

```json
{
    "active": true,
    "sub": "user-12345",
    "client_id": "my-app-id",
    "scope": "openid profile email",
    "exp": 1700000000,
    "iss": "https://auth.example.com"
}
```

This adds a network call per request but allows real-time revocation. Use it for high-security scenarios where you need immediate revocation capability.

## JWT Validation in Spring Boot

Here's how to properly validate JWTs in a Spring Boot resource server:

### Configuration

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
          # Spring auto-discovers JWKS endpoint from .well-known/openid-configuration
```

### Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasAuthority("SCOPE_admin")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthConverter() {
        JwtGrantedAuthoritiesConverter authoritiesConverter = new JwtGrantedAuthoritiesConverter();
        authoritiesConverter.setAuthoritiesClaimName("roles");
        authoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(authoritiesConverter);
        return converter;
    }
}
```

### Custom JWT Validator

Sometimes you need validation beyond what Spring provides:

```java
@Component
public class CustomJwtValidator implements OAuth2TokenValidator<Jwt> {

    private final String expectedAudience;

    public CustomJwtValidator(@Value("${app.oauth2.audience}") String expectedAudience) {
        this.expectedAudience = expectedAudience;
    }

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        List<String> audience = jwt.getAudience();

        if (audience == null || !audience.contains(expectedAudience)) {
            OAuth2Error error = new OAuth2Error("invalid_token",
                    "Token not intended for this audience", null);
            return OAuth2TokenValidatorResult.failure(error);
        }

        // Check custom claims
        String tenantId = jwt.getClaimAsString("tenant_id");
        if (tenantId == null) {
            OAuth2Error error = new OAuth2Error("invalid_token",
                    "Missing required tenant_id claim", null);
            return OAuth2TokenValidatorResult.failure(error);
        }

        return OAuth2TokenValidatorResult.success();
    }
}
```

### Register Custom Validator

```java
@Bean
public JwtDecoder jwtDecoder(CustomJwtValidator customValidator) {
    NimbusJwtDecoder decoder = NimbusJwtDecoder
            .withJwkSetUri("https://auth.example.com/.well-known/jwks.json")
            .build();

    OAuth2TokenValidator<Jwt> defaultValidator = JwtValidators
            .createDefaultWithIssuer("https://auth.example.com");

    OAuth2TokenValidator<Jwt> combined = new DelegatingOAuth2TokenValidator<>(
            defaultValidator, customValidator);

    decoder.setJwtValidator(combined);
    return decoder;
}
```

## What Your JWT Validation Must Check

Here's the checklist I enforce on every project:

- **Signature** — Verify using the issuer's public key (fetched from JWKS endpoint)
- **Expiration (exp)** — Reject expired tokens. Allow a small clock skew (30 seconds max)
- **Issuer (iss)** — Must match your expected authorization server
- **Audience (aud)** — Must include your service's identifier
- **Not Before (nbf)** — If present, token isn't valid before this time
- **Algorithm** — Reject "none" algorithm. Only accept expected algorithms (RS256, ES256)

Never trust a JWT without validating all of these. I've seen production systems that only checked expiration — everything else was blindly trusted.

## Common Security Mistakes

**Storing tokens in localStorage** — Accessible to any XSS attack. Use HttpOnly cookies or in-memory storage with refresh token rotation.

**Not validating the state parameter** — Opens you to CSRF attacks. Generate a cryptographic random state, store it in the session, verify on callback.

**Accepting tokens from any issuer** — Always validate the `iss` claim. A misconfigured resource server that accepts any valid JWT is an open door.

**Long-lived access tokens** — I've seen 30-day access tokens. If it leaks, you're exposed for a month. Keep access tokens under 1 hour.

**Not implementing token revocation** — When an employee leaves or a session is compromised, you need immediate revocation. For JWTs, this means either short expiry + refresh token revocation, or a token blacklist checked on every request.

**Exposing client secrets in SPAs** — If your JavaScript contains a client secret, it's not a secret anymore. Use PKCE for public clients.

## The Full Picture: A Real-World Flow

Here's how I wire this up in production Spring Boot applications:

1. **User clicks Login** → Frontend redirects to auth server with PKCE challenge
2. **User authenticates** → Auth server redirects back with authorization code
3. **Backend exchanges code** → Receives access token (JWT) + refresh token
4. **Backend sets HttpOnly cookie** → Contains a session reference, not the raw token
5. **Each API request** → Cookie sent automatically, backend validates the associated JWT
6. **Token expires** → Backend uses refresh token to get a new access token (transparent to user)
7. **Refresh token expires** → User redirected to login again
8. **User logs out** → Refresh token revoked, session cleared, cookie deleted

This gives you security (tokens never in browser JS), convenience (silent refresh), and revocability (kill refresh token = kill session).

## Final Advice

OAuth2 is not complicated — it's just detailed. Every piece exists because of a specific attack vector or usability requirement. When you understand the "why" behind each component, debugging becomes straightforward.

My biggest piece of advice: set up a local authorization server (Keycloak or Spring Authorization Server) and trace every HTTP request with Wireshark or your browser's network tab. Seeing the actual bytes exchanged teaches you more than any diagram.
