  # Security — Lead .NET Interview Prep

> **Audience:** Lead .NET Software Engineer interviews  
> **Focus:** JWT, OAuth 2.0, OIDC, HTTPS/TLS, OWASP Top 10, XSS, CSRF, SQL Injection, Secrets, Encryption, Input Validation

---

## Table of Contents

1. [JWT Deep Dive](#1-jwt-deep-dive)
2. [OAuth 2.0 Flows](#2-oauth-20-flows)
3. [OpenID Connect (OIDC)](#3-openid-connect-oidc)
4. [HTTPS and TLS](#4-https-and-tls)
5. [OWASP Top 10 (.NET Specifics)](#5-owasp-top-10-net-specifics)
6. [XSS — Cross-Site Scripting](#6-xss--cross-site-scripting)
7. [CSRF — Cross-Site Request Forgery](#7-csrf--cross-site-request-forgery)
8. [SQL Injection Prevention](#8-sql-injection-prevention)
9. [Secrets Management](#9-secrets-management)
10. [Encryption and Hashing](#10-encryption-and-hashing)
11. [Input Validation](#11-input-validation)
12. [Interview Q&A Rapid Fire](#12-interview-qa-rapid-fire)

---

## 1. JWT Deep Dive

### Structure

A JWT is three Base64URL-encoded parts joined by dots:

```
header.payload.signature
```

**Example (decoded):**

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9   <- header
.eyJzdWIiOiJ1c2VyMTIzIiwiaXNzIjoiaHR0cHM6Ly9hdXRoLm15YXBwLmNvbSIsImF1ZCI6ImFwaS5teWFwcC5jb20iLCJleHAiOjE3MjMwMDAwMDAsImlhdCI6MTcyMjk5NjQwMH0
.SIGNATURE
```

---

### Header

```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```

| Algorithm | Type | Key | Use Case |
|-----------|------|-----|----------|
| HS256 | Symmetric | Shared secret | Single-service internal |
| RS256 | Asymmetric | Private/Public key pair | Distributed systems, third-party auth |
| ES256 | Asymmetric (Elliptic Curve) | Private/Public key pair | Mobile, IoT (smaller keys) |

> **Pitfall:** HS256 means your API must know the signing secret. RS256 means your API only needs the public key — safer for distributed systems.

---

### Payload (Claims)

```json
{
  "sub": "user-123",        // Subject — who this token is about
  "iss": "https://auth.myapp.com",  // Issuer — who created the token
  "aud": "api.myapp.com",   // Audience — who should accept it
  "exp": 1723000000,        // Expiry (Unix timestamp)
  "iat": 1722996400,        // Issued At
  "jti": "unique-token-id", // JWT ID (for revocation)
  "roles": ["admin", "editor"],  // Custom claims
  "email": "user@example.com"
}
```

---

### Signature

For **HS256**: `HMAC-SHA256(base64url(header) + "." + base64url(payload), secret)`

For **RS256**: `RSA-SHA256(base64url(header) + "." + base64url(payload), privateKey)`

The signature prevents tampering. If anyone changes the payload, the signature becomes invalid.

---

### JWT Validation Checklist

When your API receives a JWT, validate **all** of these:

- [ ] **Signature** — verify with the correct key
- [ ] **Algorithm** — confirm `alg` matches what you expect (reject `none`)
- [ ] **Expiry (`exp`)** — token must not be expired
- [ ] **Not Before (`nbf`)** — if present, token must not be used too early
- [ ] **Issuer (`iss`)** — must match your expected issuer
- [ ] **Audience (`aud`)** — must include your API's identifier
- [ ] **Token Type** — ID token ≠ access token

---

### Common JWT Attacks

#### Attack 1: alg:none Attack

An attacker modifies the header to `"alg": "none"` and strips the signature. A vulnerable server skips validation.

```csharp
// VULNERABLE — trusting the algorithm from the token itself
var handler = new JwtSecurityTokenHandler();
var token = handler.ReadJwtToken(rawToken);
string alg = token.Header.Alg; // NEVER do this to pick your validator
```

```csharp
// SAFE — always specify the expected algorithm explicitly
var validationParameters = new TokenValidationParameters
{
    ValidateIssuerSigningKey = true,
    IssuerSigningKey = new RsaSecurityKey(publicKey),
    ValidAlgorithms = new[] { "RS256" }, // Whitelist only
    ValidateIssuer = true,
    ValidIssuer = "https://auth.myapp.com",
    ValidateAudience = true,
    ValidAudience = "api.myapp.com",
    ValidateLifetime = true
};
```

#### Attack 2: JWT Confusion Attack (HS/RS Confusion)

If the server uses RS256 but also accepts HS256, an attacker signs a token using the **public key** as the HMAC secret. The server then validates it as HS256 using the public key — and it passes.

**Fix:** Explicitly whitelist the expected algorithm. Never auto-detect.

---

### Access Token + Refresh Token Pattern

```
Client → POST /auth/login → Server
         ← { access_token: (15 min), refresh_token: (7 days) }

Client uses access_token for API calls.
When access_token expires:
Client → POST /auth/refresh { refresh_token }
         ← { access_token: (new 15 min) }
```

**Why?**
- Short-lived access tokens limit damage if stolen
- Refresh tokens can be revoked (stored server-side or in DB)
- Access tokens are stateless; refresh tokens are stateful

---

### Token Storage

| Storage | XSS Risk | CSRF Risk | Notes |
|---------|----------|-----------|-------|
| `localStorage` | HIGH | None | Accessible to JS; bad for sensitive tokens |
| `sessionStorage` | HIGH | None | Same as localStorage but tab-scoped |
| `httpOnly` Cookie | None | YES (mitigated by SameSite) | Preferred for web apps |

> **Best Practice:** Store tokens in `httpOnly; Secure; SameSite=Strict` cookies. The browser sends them automatically but JavaScript cannot read them.

---

### Real C# Code: Generate and Validate JWT

```csharp
// NuGet: Microsoft.IdentityModel.Tokens, System.IdentityModel.Tokens.Jwt

public class JwtService
{
    private readonly JwtSettings _settings;

    public JwtService(IOptions<JwtSettings> settings)
    {
        _settings = settings.Value;
    }

    // GENERATE JWT (RS256)
    public string GenerateToken(string userId, string email, IEnumerable<string> roles)
    {
        using var rsa = RSA.Create();
        rsa.ImportFromPem(_settings.PrivateKeyPem);
        var rsaKey = new RsaSecurityKey(rsa);
        var credentials = new SigningCredentials(rsaKey, SecurityAlgorithms.RsaSha256);

        var claims = new List<Claim>
        {
            new(JwtRegisteredClaimNames.Sub, userId),
            new(JwtRegisteredClaimNames.Email, email),
            new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            new(JwtRegisteredClaimNames.Iat,
                DateTimeOffset.UtcNow.ToUnixTimeSeconds().ToString(),
                ClaimValueTypes.Integer64)
        };

        foreach (var role in roles)
            claims.Add(new Claim(ClaimTypes.Role, role));

        var token = new JwtSecurityToken(
            issuer: _settings.Issuer,
            audience: _settings.Audience,
            claims: claims,
            notBefore: DateTime.UtcNow,
            expires: DateTime.UtcNow.AddMinutes(15),
            signingCredentials: credentials
        );

        return new JwtSecurityTokenHandler().WriteToken(token);
    }

    // VALIDATE JWT
    public ClaimsPrincipal? ValidateToken(string token)
    {
        using var rsa = RSA.Create();
        rsa.ImportFromPem(_settings.PublicKeyPem);
        var rsaKey = new RsaSecurityKey(rsa);

        var handler = new JwtSecurityTokenHandler();
        try
        {
            var principal = handler.ValidateToken(token, new TokenValidationParameters
            {
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = rsaKey,
                ValidAlgorithms = new[] { SecurityAlgorithms.RsaSha256 },
                ValidateIssuer = true,
                ValidIssuer = _settings.Issuer,
                ValidateAudience = true,
                ValidAudience = _settings.Audience,
                ValidateLifetime = true,
                ClockSkew = TimeSpan.FromSeconds(30) // Small tolerance only
            }, out _);

            return principal;
        }
        catch (SecurityTokenException)
        {
            return null; // Invalid token
        }
    }
}

// Program.cs — Wire up JWT Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuerSigningKey = true,
            IssuerSigningKeyResolver = (token, secToken, kid, parameters) =>
            {
                // Fetch JWKS from identity server (cached)
                return _keyResolver.GetKeys(kid);
            },
            ValidAlgorithms = new[] { "RS256" },
            ValidIssuer = "https://auth.myapp.com",
            ValidAudience = "api.myapp.com",
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30)
        };
    });
```

---

## 2. OAuth 2.0 Flows

OAuth 2.0 is an **authorization** framework. It answers: "Can application X access resource Y on behalf of user Z?"

### Flow Comparison

| Flow | Use Case | Client Secret? | Token in URL? |
|------|----------|----------------|---------------|
| Authorization Code | Web apps with backend | Yes | No |
| Authorization Code + PKCE | SPA, Mobile | No | No |
| Client Credentials | Machine-to-machine | Yes | No |
| Implicit | Deprecated | No | **Yes** (dangerous) |
| Device Flow | TV, CLI, IoT | Optional | No |

---

### Authorization Code Flow (Web Apps)

```
1. User clicks "Login" → App redirects to:
   GET https://auth.myapp.com/authorize
     ?client_id=myapp
     &redirect_uri=https://myapp.com/callback
     &response_type=code
     &scope=openid profile
     &state=random-csrf-token

2. User logs in at auth server → redirected to:
   GET https://myapp.com/callback?code=AUTH_CODE&state=random-csrf-token

3. Backend exchanges code:
   POST https://auth.myapp.com/token
     client_id=myapp
     client_secret=SECRET          <- Never expose this to browser
     code=AUTH_CODE
     grant_type=authorization_code
     redirect_uri=https://myapp.com/callback

4. Auth server returns:
   { access_token, refresh_token, id_token, expires_in }
```

**Why `state` param?** Prevents CSRF on the callback endpoint. Validate it matches what you sent.

---

### PKCE — Proof Key for Code Exchange

Used when there is **no client secret** (SPA, mobile app). PKCE proves that the entity exchanging the code is the same one that started the flow.

```
code_verifier = random 128-char string (stored locally)
code_challenge = BASE64URL(SHA256(code_verifier))

Step 1: Send code_challenge in /authorize request
Step 2: Send code_verifier in /token request
Auth server hashes verifier, compares to challenge → no interception possible
```

```csharp
// Generate PKCE pair in C#
public static (string verifier, string challenge) GeneratePkce()
{
    var bytes = RandomNumberGenerator.GetBytes(96);
    var verifier = Base64UrlEncoder.Encode(bytes);
    var hash = SHA256.HashData(Encoding.ASCII.GetBytes(verifier));
    var challenge = Base64UrlEncoder.Encode(hash);
    return (verifier, challenge);
}
```

---

### Client Credentials Flow (Machine-to-Machine)

No user involved. Service A authenticates as itself to call Service B.

```csharp
// .NET API calling downstream API using Client Credentials
// NuGet: Microsoft.Extensions.Http

// Program.cs
builder.Services.AddHttpClient<DownstreamApiClient>(client =>
{
    client.BaseAddress = new Uri("https://api.downstream.com");
})
.AddClientCredentialsTokenHandler(options =>
{
    options.Authority = "https://auth.myapp.com";
    options.ClientId = "my-service-client";
    options.ClientSecret = Environment.GetEnvironmentVariable("CLIENT_SECRET");
    options.Scope = "downstream-api";
});

// Alternatively, manual approach:
public class TokenService
{
    private readonly HttpClient _httpClient;

    public async Task<string> GetClientCredentialsTokenAsync()
    {
        var response = await _httpClient.PostAsync(
            "https://auth.myapp.com/token",
            new FormUrlEncodedContent(new[]
            {
                new KeyValuePair<string, string>("grant_type", "client_credentials"),
                new KeyValuePair<string, string>("client_id", "my-service"),
                new KeyValuePair<string, string>("client_secret",
                    Environment.GetEnvironmentVariable("CLIENT_SECRET")!),
                new KeyValuePair<string, string>("scope", "downstream-api")
            }));

        var json = await response.Content.ReadFromJsonAsync<TokenResponse>();
        return json!.AccessToken;
    }
}
```

---

### Device Flow (CLI/TV)

```
1. App → POST /device_authorization → { device_code, user_code, verification_uri }
2. App displays: "Go to https://auth.myapp.com/activate and enter: ABCD-1234"
3. App polls /token with device_code until user completes auth
4. User completes on phone/browser → app gets token
```

---

## 3. OpenID Connect (OIDC)

OAuth 2.0 handles **authorization**. OIDC adds **authentication** — it answers "Who is this user?"

### Key Differences from OAuth 2.0

| | OAuth 2.0 | OIDC |
|---|-----------|------|
| Purpose | Authorization | Authentication + Authorization |
| Token | Access Token | ID Token + Access Token |
| User Info | Not defined | ID Token claims + UserInfo endpoint |
| Standard | RFC 6749 | OpenID Foundation spec |

### ID Token vs Access Token

```json
// ID Token — tells YOUR APP who the user is (don't send to APIs)
{
  "sub": "user-123",
  "name": "Jane Doe",
  "email": "jane@example.com",
  "email_verified": true,
  "iss": "https://accounts.google.com",
  "aud": "your-client-id",       // YOUR app
  "exp": 1723000000,
  "iat": 1722996400
}

// Access Token — send to APIs (opaque or JWT, depends on provider)
// APIs validate it, check scopes for what the user can do
```

### Discovery Endpoint

Every OIDC provider exposes:

```
GET https://auth.myapp.com/.well-known/openid-configuration

Returns:
{
  "issuer": "https://auth.myapp.com",
  "authorization_endpoint": "https://auth.myapp.com/authorize",
  "token_endpoint": "https://auth.myapp.com/token",
  "userinfo_endpoint": "https://auth.myapp.com/userinfo",
  "jwks_uri": "https://auth.myapp.com/.well-known/jwks.json",
  "scopes_supported": ["openid", "profile", "email"],
  "id_token_signing_alg_values_supported": ["RS256"]
}
```

### SSO with Azure AD in ASP.NET Core

```csharp
// Program.cs
builder.Services.AddAuthentication(options =>
{
    options.DefaultScheme = CookieAuthenticationDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = OpenIdConnectDefaults.AuthenticationScheme;
})
.AddCookie()
.AddOpenIdConnect(options =>
{
    options.Authority = "https://login.microsoftonline.com/{tenantId}/v2.0";
    options.ClientId = "your-client-id";
    options.ClientSecret = Environment.GetEnvironmentVariable("AZURE_CLIENT_SECRET");
    options.ResponseType = "code";
    options.Scope.Add("openid");
    options.Scope.Add("profile");
    options.Scope.Add("email");
    options.SaveTokens = true; // Saves access + refresh token in cookie
    options.GetClaimsFromUserInfoEndpoint = true;
    options.TokenValidationParameters = new TokenValidationParameters
    {
        NameClaimType = "name",
        RoleClaimType = "roles"
    };
});
```

---

## 4. HTTPS and TLS

### TLS 1.2 vs TLS 1.3

| Feature | TLS 1.2 | TLS 1.3 |
|---------|---------|---------|
| Handshake RTTs | 2 | 1 (or 0-RTT) |
| Cipher suites | Many (some weak) | Only modern (AES-GCM, ChaCha20) |
| Perfect Forward Secrecy | Optional | Mandatory |
| Session resumption | Session ID / tickets | PSK |
| Security | Good | Better — removes deprecated algorithms |
| .NET support | Full | .NET 5+ / Windows 10+ |

> **Mandate TLS 1.2+ minimum.** TLS 1.0 and 1.1 are deprecated and vulnerable (POODLE, BEAST).

---

### Certificates

- **CA-signed:** Trusted by browsers. Use Let's Encrypt (free) or commercial CAs for production.
- **Self-signed:** Dev/testing only. Browsers will warn users.
- **Let's Encrypt:** Free, automated, 90-day renewal via ACME protocol.

---

### HSTS — HTTP Strict Transport Security

Tells browsers to **only** use HTTPS for your domain, even if user types `http://`.

```csharp
// Program.cs
if (!app.Environment.IsDevelopment())
{
    app.UseHsts(); // Sends: Strict-Transport-Security: max-age=31536000
}
app.UseHttpsRedirection();
```

**Preloading:** Submit your domain to https://hstspreload.org — browsers will hardcode HTTPS for your domain even on first visit.

```
Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
```

---

### Certificate Pinning

Hardcoding the expected certificate or public key in the client. Prevents MITM even if a rogue CA signs a cert for your domain.

```csharp
// HttpClientHandler with certificate pinning
var handler = new HttpClientHandler();
handler.ServerCertificateCustomValidationCallback = (request, cert, chain, errors) =>
{
    if (errors != SslPolicyErrors.None)
        return false;

    // Pin to specific certificate thumbprint
    var expectedThumbprint = "A1B2C3D4..."; // SHA-1 thumbprint
    return cert?.GetCertHashString() == expectedThumbprint;
};

var client = new HttpClient(handler);
```

> **Pitfall:** Certificate pinning can break your app when certs rotate. Use public key pinning or maintain a pin list with backup pins.

---

### Enforce HTTPS in ASP.NET Core

```csharp
// Program.cs — Minimal API
var builder = WebApplication.CreateBuilder(args);

// Configure Kestrel for HTTPS
builder.WebHost.ConfigureKestrel(options =>
{
    options.Listen(IPAddress.Any, 5000);
    options.Listen(IPAddress.Any, 5001, listenOptions =>
    {
        listenOptions.UseHttps("certificate.pfx", "password");
    });
});

var app = builder.Build();

// Redirect HTTP to HTTPS
app.UseHttpsRedirection();

// HSTS in production
if (!app.Environment.IsDevelopment())
{
    app.UseHsts();
}

// Security headers
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("X-Content-Type-Options", "nosniff");
    context.Response.Headers.Add("X-Frame-Options", "DENY");
    context.Response.Headers.Add("X-XSS-Protection", "1; mode=block");
    context.Response.Headers.Add("Referrer-Policy", "strict-origin-when-cross-origin");
    await next();
});
```

---

## 5. OWASP Top 10 (.NET Specifics)

### A01: Broken Access Control

The most common and impactful vulnerability. Authorization checks missing or bypassable.

**Vulnerable:**
```csharp
// VULNERABLE — only checks if user is logged in, not if they OWN the document
[Authorize]
[HttpGet("documents/{id}")]
public async Task<IActionResult> GetDocument(int id)
{
    var doc = await _db.Documents.FindAsync(id);
    return Ok(doc); // Any authenticated user can see any document!
}
```

**Safe — Resource-Based Authorization:**
```csharp
// SAFE — checks ownership
[Authorize]
[HttpGet("documents/{id}")]
public async Task<IActionResult> GetDocument(int id)
{
    var doc = await _db.Documents.FindAsync(id);
    if (doc == null) return NotFound();

    // Resource-based authorization
    var authResult = await _authorizationService.AuthorizeAsync(
        User, doc, DocumentOperations.Read);

    if (!authResult.Succeeded)
        return Forbid(); // 403, not 401

    return Ok(doc);
}

// Authorization handler
public class DocumentAuthorizationHandler
    : AuthorizationHandler<OperationAuthorizationRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OperationAuthorizationRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirstValue(ClaimTypes.NameIdentifier);

        if (requirement == DocumentOperations.Read &&
            resource.OwnerId == userId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

**OWASP A01 Fixes:**
- Deny by default — start with no access
- Apply authorization at every layer (controller, service, data)
- Log access denials
- Never rely solely on client-side checks

---

### A02: Cryptographic Failures

Previously "Sensitive Data Exposure." The root cause is bad cryptography choices.

**Vulnerable — Storing Passwords in Plaintext or with MD5:**
```csharp
// VULNERABLE — NEVER do this
public async Task CreateUser(string email, string password)
{
    var user = new User
    {
        Email = email,
        PasswordHash = MD5.HashData(Encoding.UTF8.GetBytes(password))
                          .ToHexString() // MD5 is trivially crackable
    };
    await _db.Users.AddAsync(user);
}
```

**Safe — BCrypt or ASP.NET Core Identity:**
```csharp
// SAFE — using BCrypt.Net (NuGet: BCrypt.Net-Next)
public async Task CreateUser(string email, string password)
{
    var user = new User
    {
        Email = email,
        // Work factor 12 = ~300ms per hash (adjust as hardware improves)
        PasswordHash = BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12)
    };
    await _db.Users.AddAsync(user);
}

public bool VerifyPassword(string password, string hash)
    => BCrypt.Net.BCrypt.Verify(password, hash);

// ALSO SAFE — ASP.NET Core Identity built-in (uses PBKDF2)
var hasher = new PasswordHasher<User>();
string hash = hasher.HashPassword(user, password);
PasswordVerificationResult result = hasher.VerifyHashedPassword(user, hash, password);
```

**Why not MD5/SHA1 for passwords?**
- No salt by default → rainbow table attacks
- Too fast → brute-force billions/sec on GPUs
- Use BCrypt, Argon2, or PBKDF2 — they are intentionally slow

---

### A03: SQL Injection

One of the oldest and most critical vulnerabilities.

**Vulnerable:**
```csharp
// VULNERABLE — string interpolation in SQL
public async Task<User?> GetUser(string username)
{
    var sql = $"SELECT * FROM Users WHERE Username = '{username}'";
    // username = "admin' OR '1'='1" → returns all users
    // username = "'; DROP TABLE Users;--" → deletes table
    return await _db.Database
        .SqlQueryRaw<User>(sql)
        .FirstOrDefaultAsync();
}
```

**Safe — Parameterized:**
```csharp
// SAFE — parameterized query (EF Core)
public async Task<User?> GetUser(string username)
{
    return await _db.Users
        .Where(u => u.Username == username) // EF Core LINQ — safe by default
        .FirstOrDefaultAsync();
}

// SAFE — raw SQL with parameters
public async Task<User?> GetUserRaw(string username)
{
    return await _db.Database
        .SqlQueryRaw<User>(
            "SELECT * FROM Users WHERE Username = {0}", // Parameterized
            username)
        .FirstOrDefaultAsync();
}

// SAFE — SqlCommand
public async Task<DataTable> GetUserAdo(string username)
{
    using var conn = new SqlConnection(_connectionString);
    using var cmd = new SqlCommand(
        "SELECT * FROM Users WHERE Username = @Username", conn);
    cmd.Parameters.AddWithValue("@Username", username);
    // OR: cmd.Parameters.Add("@Username", SqlDbType.NVarChar, 100).Value = username;
    await conn.OpenAsync();
    // ...
}
```

> **EF Core Pitfall:** `ExecuteSqlRaw` with string interpolation is vulnerable. Use `ExecuteSqlInterpolated` (uses `FormattableString` with proper parameterization) or `ExecuteSqlRaw` with SqlParameter objects.

```csharp
// VULNERABLE
await _db.Database.ExecuteSqlRaw($"DELETE FROM Logs WHERE UserId = {userId}");

// SAFE
await _db.Database.ExecuteSqlInterpolated($"DELETE FROM Logs WHERE UserId = {userId}");
// OR
await _db.Database.ExecuteSqlRaw(
    "DELETE FROM Logs WHERE UserId = @id",
    new SqlParameter("@id", userId));
```

---

### A05: Security Misconfiguration

**Vulnerable — Exposing stack traces in production:**
```csharp
// VULNERABLE — shows full stack trace to users in prod
app.UseDeveloperExceptionPage(); // Only for development!
```

**Safe:**
```csharp
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
}
else
{
    app.UseExceptionHandler("/error"); // Generic error page
    app.UseHsts();
}

// Global error handler — never expose details
app.Map("/error", (HttpContext context) =>
{
    var exceptionFeature = context.Features.Get<IExceptionHandlerFeature>();
    // Log the real error, return generic message
    _logger.LogError(exceptionFeature?.Error, "Unhandled exception");
    return Results.Problem("An unexpected error occurred.");
});
```

**Other misconfigurations:**
- Default admin credentials not changed
- Swagger/debug endpoints enabled in production
- CORS policy too permissive (`AllowAnyOrigin`)
- Directory listing enabled on web servers

---

### A06: Vulnerable Components

```bash
# Check for known vulnerable NuGet packages
dotnet list package --vulnerable

# Check for outdated packages
dotnet list package --outdated

# In CI pipeline — fail build if vulnerable packages found
dotnet list package --vulnerable --format json | jq '.projects[].frameworks[].topLevelPackages[] | select(.vulnerabilities != null)'
```

- Enable **Dependabot** in GitHub (auto-PRs for security updates)
- Use **NuGet Audit** (built into .NET 8 SDK) — runs automatically on `dotnet restore`
- Pin package versions in `packages.lock.json`

---

### A07: Authentication Failures

**Vulnerable — No rate limiting on login:**
```csharp
// VULNERABLE — allows unlimited login attempts
[HttpPost("login")]
public async Task<IActionResult> Login(LoginRequest request)
{
    var user = await _userManager.FindByEmailAsync(request.Email);
    if (user != null && await _userManager.CheckPasswordAsync(user, request.Password))
        return Ok(GenerateToken(user));
    return Unauthorized();
}
```

**Safe — With lockout and rate limiting:**
```csharp
// Program.cs — Configure Identity lockout
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(options =>
{
    options.Lockout.DefaultLockoutTimeSpan = TimeSpan.FromMinutes(15);
    options.Lockout.MaxFailedAccessAttempts = 5;
    options.Lockout.AllowedForNewUsers = true;

    options.Password.RequiredLength = 12;
    options.Password.RequireUppercase = true;
    options.Password.RequireDigit = true;
    options.Password.RequireNonAlphanumeric = true;
});

// Login controller — uses Identity's built-in lockout
[HttpPost("login")]
public async Task<IActionResult> Login(LoginRequest request)
{
    var result = await _signInManager.PasswordSignInAsync(
        request.Email,
        request.Password,
        isPersistent: false,
        lockoutOnFailure: true); // Increments lockout counter

    if (result.IsLockedOut)
        return StatusCode(429, "Account locked. Try again in 15 minutes.");

    if (!result.Succeeded)
        return Unauthorized("Invalid credentials."); // Generic message — no user enumeration

    return Ok(GenerateToken(request.Email));
}

// Rate limiting (Program.cs — .NET 7+)
builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("login", config =>
    {
        config.PermitLimit = 5;
        config.Window = TimeSpan.FromMinutes(1);
        config.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
        config.QueueLimit = 0;
    });
});

app.UseRateLimiter();
```

---

### A08: Software Integrity Failures

- Supply chain attacks (malicious packages injected into registries)
- **Fix:** Use NuGet package signing, lock files (`packages.lock.json`), verify checksums
- Internal NuGet proxy (Artifactory, Azure Artifacts) — vet packages before use
- SBOM (Software Bill of Materials) — track all dependencies

---

### A09: Logging Failures

```csharp
// SAFE — Log security events, NOT sensitive data
public class SecurityAuditLogger
{
    private readonly ILogger<SecurityAuditLogger> _logger;

    public void LogFailedLogin(string email, string ipAddress)
    {
        // DO log: what happened, who, when, from where
        _logger.LogWarning(
            "Failed login attempt. Email={Email} IP={IpAddress} Time={Time}",
            email,  // Email is OK if PII policy allows
            ipAddress,
            DateTimeOffset.UtcNow);
    }

    public void LogSuccessfulLogin(string userId, string ipAddress)
    {
        _logger.LogInformation(
            "Successful login. UserId={UserId} IP={IpAddress}",
            userId, ipAddress);
    }

    // NEVER log these
    public void BadExample(string password, string token)
    {
        _logger.LogDebug("User password: {Password}", password);   // NEVER
        _logger.LogDebug("Access token: {Token}", token);          // NEVER
        _logger.LogDebug("Credit card: {CC}", creditCardNumber);   // NEVER
    }
}
```

**Use structured logging + centralized SIEM:**
- Serilog / NLog → CloudWatch / ELK / Splunk / Azure Monitor
- Set alerts on: 5+ failed logins from same IP, privilege escalation, admin access outside hours

---

### A10: SSRF (Server-Side Request Forgery)

Server is tricked into making requests to internal resources.

**Vulnerable:**
```csharp
// VULNERABLE — user controls the URL, can hit internal metadata endpoints
[HttpGet("fetch")]
public async Task<IActionResult> FetchUrl([FromQuery] string url)
{
    var content = await _httpClient.GetStringAsync(url);
    // Attacker sends: url=http://169.254.169.254/latest/meta-data/
    // (AWS instance metadata — gets IAM credentials!)
    return Ok(content);
}
```

**Safe — URL Allowlist:**
```csharp
// SAFE — validate against allowlist before making request
private static readonly HashSet<string> _allowedHosts = new(StringComparer.OrdinalIgnoreCase)
{
    "api.trusted-partner.com",
    "data.public-source.com"
};

[HttpGet("fetch")]
public async Task<IActionResult> FetchUrl([FromQuery] string url)
{
    if (!Uri.TryCreate(url, UriKind.Absolute, out var uri))
        return BadRequest("Invalid URL");

    if (uri.Scheme != "https")
        return BadRequest("Only HTTPS allowed");

    if (!_allowedHosts.Contains(uri.Host))
        return BadRequest("Host not allowed");

    // Also block private IP ranges
    var addresses = await Dns.GetHostAddressesAsync(uri.Host);
    if (addresses.Any(IsPrivateIp))
        return BadRequest("Internal addresses not allowed");

    var content = await _httpClient.GetStringAsync(uri);
    return Ok(content);
}

private static bool IsPrivateIp(IPAddress ip)
{
    var bytes = ip.GetAddressBytes();
    return bytes[0] == 10
        || (bytes[0] == 172 && bytes[1] >= 16 && bytes[1] <= 31)
        || (bytes[0] == 192 && bytes[1] == 168)
        || (bytes[0] == 169 && bytes[1] == 254); // Link-local / metadata
}
```

---

## 6. XSS — Cross-Site Scripting

### Types

| Type | Description | Example |
|------|-------------|---------|
| **Reflected** | Payload in URL reflected in response | `?search=<script>alert(1)</script>` |
| **Stored** | Payload saved in DB, shown to other users | Comment: `<script>stealCookies()</script>` |
| **DOM-based** | Payload executed by client-side JS | `document.write(location.hash)` |

### Razor Auto-Encodes (Safe by Default)

```razor
@* SAFE — Razor HTML-encodes by default *@
<p>Welcome, @Model.Username</p>
@* Outputs: Welcome, &lt;script&gt;alert(1)&lt;/script&gt; *@

@* UNSAFE — @Html.Raw bypasses encoding *@
<p>@Html.Raw(Model.UserBio)</p>
@* If bio contains <script>, it executes! *@
```

**Only use `@Html.Raw()` for:**
- Content you generated yourself (never user input)
- Content processed by a safe HTML sanitizer

```csharp
// Safe HTML sanitization (NuGet: Ganss.Xss — HtmlSanitizer)
var sanitizer = new HtmlSanitizer();
var safeHtml = sanitizer.Sanitize(userInput);
// Now safe to use with @Html.Raw(safeHtml)
```

### Content Security Policy (CSP)

```csharp
// Add CSP header to block inline scripts
app.Use(async (context, next) =>
{
    context.Response.Headers.Add("Content-Security-Policy",
        "default-src 'self'; " +
        "script-src 'self' 'nonce-{random}' https://cdn.trusted.com; " +
        "style-src 'self' https://fonts.googleapis.com; " +
        "img-src 'self' data: https:; " +
        "font-src 'self' https://fonts.gstatic.com; " +
        "connect-src 'self' https://api.myapp.com; " +
        "frame-ancestors 'none';");
    await next();
});
```

---

## 7. CSRF — Cross-Site Request Forgery

An attacker's site tricks the victim's browser into submitting a request to your site using the victim's cookies.

**Attack:**
```html
<!-- Attacker's site — victim visits this page while logged into bank.com -->
<form action="https://bank.com/transfer" method="POST">
  <input type="hidden" name="to" value="attacker-account" />
  <input type="hidden" name="amount" value="10000" />
</form>
<script>document.forms[0].submit();</script>
```

### ASP.NET Core CSRF Protection

```csharp
// Program.cs — Enable antiforgery globally
builder.Services.AddAntiforgery(options =>
{
    options.HeaderName = "X-XSRF-TOKEN"; // For SPA APIs
    options.Cookie.Name = "XSRF-TOKEN";
    options.Cookie.HttpOnly = false; // SPA needs to read and set header
    options.Cookie.SameSite = SameSiteMode.Strict;
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
});

// Controller — MVC forms
[AutoValidateAntiforgeryToken] // Apply to all POST/PUT/PATCH/DELETE
public class AccountController : Controller
{
    [HttpPost]
    [ValidateAntiForgeryToken] // Per-action alternative
    public async Task<IActionResult> Transfer(TransferRequest request)
    {
        // ...
    }
}

// Razor form — include token
<form asp-action="Transfer" method="post">
    @Html.AntiForgeryToken()  // Adds hidden input
    <!-- ... -->
</form>
```

### SameSite Cookie Attribute

```csharp
// Configure authentication cookie
builder.Services.ConfigureApplicationCookie(options =>
{
    options.Cookie.SameSite = SameSiteMode.Strict; // Best CSRF protection
    // Strict: cookie not sent on cross-site requests AT ALL
    // Lax: cookie sent on top-level navigation (links), not forms
    // None: sent cross-site (requires Secure)
    options.Cookie.SecurePolicy = CookieSecurePolicy.Always;
    options.Cookie.HttpOnly = true;
});
```

### API CSRF Protection (No Cookies)

For JWT-based APIs (tokens in Authorization header):
- CSRF is not applicable — tokens aren't automatically sent by browsers
- The browser only automatically sends cookies, not Authorization headers

For cookie-authenticated APIs:
- Use custom headers (`X-Requested-With: XMLHttpRequest`)
- Validate Origin/Referer header
- Use the antiforgery token in a header (`X-XSRF-TOKEN`)

---

## 8. SQL Injection Prevention

### Summary of Approaches

| Approach | Safety | Notes |
|----------|--------|-------|
| EF Core LINQ queries | Safe | Parameterized automatically |
| `ExecuteSqlInterpolated` | Safe | Uses FormattableString |
| `ExecuteSqlRaw` + SqlParameter | Safe | Manual but correct |
| `ExecuteSqlRaw` + string concat | **UNSAFE** | Never do this |
| Stored Procedures | Mostly safe | Can still be vulnerable if SP uses dynamic SQL |

```csharp
// VULNERABLE — Do NOT do any of these
var sql1 = $"SELECT * FROM Users WHERE Id = {id}";
var sql2 = "SELECT * FROM Users WHERE Name = '" + name + "'";
var sql3 = string.Format("SELECT * FROM Users WHERE Email = '{0}'", email);

await _db.Database.ExecuteSqlRaw(sql1); // VULNERABLE
await _db.Database.ExecuteSqlRaw(sql2); // VULNERABLE

// SAFE approaches
// 1. EF Core LINQ (always use this for standard queries)
var user = await _db.Users.Where(u => u.Id == id).FirstOrDefaultAsync();

// 2. ExecuteSqlInterpolated (EF Core 3+) — looks like interpolation but is parameterized
await _db.Database.ExecuteSqlInterpolated($"UPDATE Users SET LastLogin = {DateTime.UtcNow} WHERE Id = {id}");

// 3. ExecuteSqlRaw with SqlParameter
await _db.Database.ExecuteSqlRaw(
    "UPDATE Users SET LastLogin = @date WHERE Id = @id",
    new SqlParameter("@date", DateTime.UtcNow),
    new SqlParameter("@id", id));

// 4. Stored procedure (safe if SP itself is parameterized)
await _db.Database.ExecuteSqlRaw("EXEC UpdateLastLogin @UserId", 
    new SqlParameter("@UserId", userId));
```

---

## 9. Secrets Management

### Never Do This

```csharp
// appsettings.json — NEVER commit secrets here
{
    "ConnectionStrings": {
        "DefaultConnection": "Server=prod.db.com;Password=SuperSecret123!" // WRONG
    },
    "Jwt": {
        "Secret": "my-hardcoded-secret" // WRONG
    }
}

// Code — NEVER hardcode
private const string ApiKey = "sk-live-abc123"; // WRONG
```

### Development: User Secrets

```bash
# .NET user secrets — stored outside project directory
dotnet user-secrets init
dotnet user-secrets set "Jwt:Secret" "dev-only-secret"
dotnet user-secrets set "ConnectionStrings:DefaultConnection" "Server=localhost;..."
```

```csharp
// Automatically read in development
builder.Configuration.AddUserSecrets<Program>(optional: true);
```

### Production: AWS Secrets Manager

```csharp
// NuGet: Amazon.Extensions.Configuration.SystemsManager
// or: AWSSDK.SecretsManager

builder.Configuration.AddSecretsManager(configurator: secretsManagerOptions =>
{
    secretsManagerOptions.SecretFilter = entry =>
        entry.Name.StartsWith("MyApp/");

    secretsManagerOptions.KeyGenerator = (entry, key) =>
        key.Replace("MyApp/", "").Replace("/", ":");

    secretsManagerOptions.PollingInterval = TimeSpan.FromHours(1); // Auto-refresh
});

// In code — same IConfiguration interface
string connectionString = builder.Configuration["ConnectionStrings:DefaultConnection"];
```

### Production: Azure Key Vault

```csharp
// NuGet: Azure.Extensions.AspNetCore.Configuration.Secrets
builder.Configuration.AddAzureKeyVault(
    new Uri($"https://{vaultName}.vault.azure.net/"),
    new DefaultAzureCredential()); // Uses managed identity in Azure
```

### Rotation Strategy

1. Store secrets with version identifiers
2. Deploy new version alongside old
3. Rotate — update secret value in vault
4. Monitor — ensure old version is no longer used
5. Remove old version

### Scanning for Leaked Secrets

```bash
# git-secrets — prevents committing secrets
git secrets --install
git secrets --register-aws

# truffleHog — scan git history
trufflehog git https://github.com/org/repo.git

# GitHub Advanced Security — secret scanning (built-in)
# Detects tokens from 100+ providers automatically
```

---

## 10. Encryption and Hashing

### Hashing vs Encryption

| | Hashing | Encryption |
|---|---------|------------|
| Reversible? | No (one-way) | Yes (with key) |
| Use for | Passwords, integrity checks | Data you need to decrypt |
| Examples | BCrypt, SHA-256, Argon2 | AES-256, RSA |

### Password Hashing

```csharp
// OPTION 1: BCrypt (recommended for new projects)
// NuGet: BCrypt.Net-Next
string hash = BCrypt.Net.BCrypt.HashPassword(password, workFactor: 12);
bool valid = BCrypt.Net.BCrypt.Verify(password, hash);
// Hash looks like: $2a$12$N9qo8uLOickgx2ZMRZoMyeIjZAgcfl7p92ldGxad68LJZdL17lhWy

// OPTION 2: ASP.NET Core Identity PasswordHasher (PBKDF2)
var hasher = new PasswordHasher<IdentityUser>();
string hash = hasher.HashPassword(user, password);
var result = hasher.VerifyHashedPassword(user, hash, password);
// result: Succeeded, Failed, SuccessRehashNeeded

// OPTION 3: Argon2 (memory-hard — best against GPU attacks)
// NuGet: Isopoh.Cryptography.Argon2
```

### Symmetric Encryption: AES-256

```csharp
// AES-256-GCM (authenticated encryption — detects tampering)
public class AesEncryption
{
    public static (byte[] ciphertext, byte[] nonce, byte[] tag) Encrypt(
        byte[] plaintext, byte[] key)
    {
        var nonce = RandomNumberGenerator.GetBytes(12); // 96-bit nonce for GCM
        var ciphertext = new byte[plaintext.Length];
        var tag = new byte[16];

        using var aes = new AesGcm(key, 16);
        aes.Encrypt(nonce, plaintext, ciphertext, tag);

        return (ciphertext, nonce, tag);
    }

    public static byte[] Decrypt(
        byte[] ciphertext, byte[] key, byte[] nonce, byte[] tag)
    {
        var plaintext = new byte[ciphertext.Length];
        using var aes = new AesGcm(key, 16);
        aes.Decrypt(nonce, ciphertext, tag, plaintext);
        // Throws if tag doesn't match (tampering detected)
        return plaintext;
    }
}
```

### Asymmetric Encryption: RSA

```csharp
// RSA — use for key exchange or small data; not for bulk data
public class RsaEncryption
{
    public static (string publicKey, string privateKey) GenerateKeyPair()
    {
        using var rsa = RSA.Create(2048);
        return (
            rsa.ExportRSAPublicKeyPem(),
            rsa.ExportRSAPrivateKeyPem()
        );
    }

    public static byte[] Encrypt(byte[] data, string publicKeyPem)
    {
        using var rsa = RSA.Create();
        rsa.ImportFromPem(publicKeyPem);
        return rsa.Encrypt(data, RSAEncryptionPadding.OaepSHA256);
    }

    public static byte[] Decrypt(byte[] ciphertext, string privateKeyPem)
    {
        using var rsa = RSA.Create();
        rsa.ImportFromPem(privateKeyPem);
        return rsa.Decrypt(ciphertext, RSAEncryptionPadding.OaepSHA256);
    }
}
```

### Data At-Rest and In-Transit

| Layer | Mechanism |
|-------|-----------|
| SQL Server at-rest | Transparent Data Encryption (TDE) — automatic |
| S3 at-rest | SSE-S3, SSE-KMS, SSE-C |
| Application-level | Encrypt sensitive columns before storing |
| In-transit | TLS everywhere |
| Field-level | AES-256 encrypt PII fields (SSN, credit card) |

---

## 11. Input Validation

### Whitelist vs Blacklist

```csharp
// BLACKLIST — trying to block bad input (always loses)
if (input.Contains("'") || input.Contains("--") || input.Contains("DROP"))
    return BadRequest(); // Attacker will find bypass

// WHITELIST — define exactly what IS allowed (correct approach)
if (!Regex.IsMatch(username, @"^[a-zA-Z0-9_]{3,20}$"))
    return BadRequest("Username must be 3-20 alphanumeric characters");
```

### DataAnnotations

```csharp
public class CreateUserRequest
{
    [Required]
    [EmailAddress]
    [MaxLength(254)] // RFC 5321 email max length
    public string Email { get; set; } = null!;

    [Required]
    [MinLength(12)]
    [MaxLength(100)]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[^\da-zA-Z]).+$",
        ErrorMessage = "Password must have upper, lower, digit, and special char")]
    public string Password { get; set; } = null!;

    [Required]
    [Range(1, 120)]
    public int Age { get; set; }

    [Url]
    public string? ProfileUrl { get; set; }
}

// Controller — ModelState is automatically validated with [ApiController]
[ApiController]
[HttpPost]
public async Task<IActionResult> CreateUser(CreateUserRequest request)
{
    // If validation fails, 400 Bad Request is returned automatically
    // ...
}
```

### FluentValidation

```csharp
// NuGet: FluentValidation.AspNetCore
public class CreateUserValidator : AbstractValidator<CreateUserRequest>
{
    public CreateUserValidator()
    {
        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MaximumLength(254)
            .MustAsync(BeUniqueEmail)
            .WithMessage("Email already in use");

        RuleFor(x => x.Password)
            .NotEmpty()
            .MinimumLength(12)
            .Must(HaveComplexity)
            .WithMessage("Password too weak");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120);
    }

    private async Task<bool> BeUniqueEmail(string email, CancellationToken ct)
        => !await _db.Users.AnyAsync(u => u.Email == email, ct);

    private bool HaveComplexity(string password)
        => password.Any(char.IsUpper)
        && password.Any(char.IsLower)
        && password.Any(char.IsDigit)
        && password.Any(c => !char.IsLetterOrDigit(c));
}

// Program.cs
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<CreateUserValidator>();
```

### File Upload Security

```csharp
[HttpPost("upload")]
public async Task<IActionResult> Upload(IFormFile file)
{
    // 1. Size limit
    const long maxSize = 5 * 1024 * 1024; // 5 MB
    if (file.Length > maxSize)
        return BadRequest("File too large");

    // 2. Extension whitelist (not blacklist)
    var allowedExtensions = new[] { ".jpg", ".png", ".pdf", ".docx" };
    var extension = Path.GetExtension(file.FileName).ToLowerInvariant();
    if (!allowedExtensions.Contains(extension))
        return BadRequest("File type not allowed");

    // 3. Validate file signature (magic bytes) — extension can be spoofed
    var signatures = new Dictionary<string, byte[]>
    {
        [".jpg"] = new byte[] { 0xFF, 0xD8, 0xFF },
        [".png"] = new byte[] { 0x89, 0x50, 0x4E, 0x47 },
        [".pdf"] = new byte[] { 0x25, 0x50, 0x44, 0x46 }
    };

    using var stream = file.OpenReadStream();
    var headerBytes = new byte[4];
    await stream.ReadExactlyAsync(headerBytes);
    stream.Position = 0;

    if (signatures.TryGetValue(extension, out var expectedHeader))
    {
        if (!headerBytes.Take(expectedHeader.Length).SequenceEqual(expectedHeader))
            return BadRequest("File content does not match extension");
    }

    // 4. Generate random filename — never use client-provided filename
    var safeFileName = $"{Guid.NewGuid()}{extension}";

    // 5. Store outside web root (not accessible via URL)
    var uploadPath = Path.Combine(_config["UploadPath"]!, safeFileName);
    using var fileStream = System.IO.File.Create(uploadPath);
    await file.CopyToAsync(fileStream);

    // 6. Optionally: scan with antivirus API
    // await _virusScanner.ScanAsync(uploadPath);

    return Ok(new { fileName = safeFileName });
}
```

---

## 12. Interview Q&A Rapid Fire

**Q: What's the difference between authentication and authorization?**
> Authentication = who you are (prove identity). Authorization = what you can do (check permissions). JWT handles both: the token proves identity; claims/roles determine what you can access.

**Q: Why is `alg: none` in JWT dangerous?**
> If a library accepts `none` as a valid algorithm, an attacker can craft a token with any payload and no signature, and the server accepts it. Always whitelist the expected algorithm explicitly.

**Q: When would you choose RS256 over HS256?**
> RS256 when multiple services verify tokens — each only needs the public key. HS256 requires sharing the secret with every verifier. RS256 also allows the auth server to be the only entity that can create tokens.

**Q: What is PKCE and why does it exist?**
> SPAs and mobile apps can't safely store a client secret. PKCE replaces the client secret with a one-time cryptographic challenge, ensuring only the entity that started the auth flow can exchange the code for a token.

**Q: How does CSRF work and how does SameSite prevent it?**
> CSRF exploits the fact that browsers automatically send cookies. SameSite=Strict prevents cookies from being sent on any cross-site request. The attacker's form submission won't include the auth cookie.

**Q: What's the difference between XSS and CSRF?**
> XSS = injecting JavaScript into a page to run in victim's browser. CSRF = tricking victim's browser into making a request using their credentials. XSS attacks the client; CSRF attacks the server via the client.

**Q: Why shouldn't you store JWTs in localStorage?**
> Any JavaScript on the page can read localStorage, including injected XSS scripts. httpOnly cookies cannot be read by JavaScript at all.

**Q: What makes Argon2/BCrypt better than SHA-256 for passwords?**
> SHA-256 is designed to be fast — a GPU can compute billions/sec, enabling brute force. BCrypt/Argon2 are intentionally slow (configurable work factor), making brute force impractical. They also salt automatically.

**Q: What's the difference between TDE and column-level encryption?**
> TDE encrypts the entire database file at rest — protects against physical theft. Column-level encryption encrypts specific fields — data is encrypted even when queried by DBAs with DB access.

**Q: How do you prevent secrets from ending up in your git history?**
> Use .gitignore for secrets files, pre-commit hooks (git-secrets), scan with truffleHog, use user-secrets in dev, Key Vault/Secrets Manager in prod, and never commit .env files.

**Q: What is SSRF and how do you prevent it?**
> Server-Side Request Forgery — server makes HTTP requests on behalf of user input, potentially to internal services. Prevent with URL allowlists, blocking private IP ranges, and not making server-side requests based on user-controlled URLs.

**Q: What does "defense in depth" mean in application security?**
> Layering multiple security controls so that failure of one doesn't compromise the system. Example: input validation + parameterized queries + WAF + principle of least privilege + monitoring.

---

## OWASP Security Checklist for .NET APIs

```
Authentication & JWT
[ ] JWT signature validation enabled (not trusting alg from token)
[ ] Short expiry on access tokens (15-60 min)
[ ] Refresh token rotation implemented
[ ] Tokens stored in httpOnly cookies (not localStorage)
[ ] Algorithm explicitly whitelisted (no "none" or auto-detect)

Authorization
[ ] Authorization on every endpoint ([Authorize] or policy)
[ ] Resource-based authorization checks ownership
[ ] Deny by default policy configured
[ ] Roles/claims validated server-side only

Transport
[ ] HTTPS enforced (UseHttpsRedirection)
[ ] HSTS header configured
[ ] TLS 1.2+ minimum (1.0/1.1 disabled)
[ ] Security headers set (X-Content-Type-Options, X-Frame-Options, CSP)

Input & Output
[ ] All input validated with whitelist approach
[ ] Parameterized queries everywhere (no string concatenation in SQL)
[ ] HTML output encoded (Razor default; no raw Html.Raw on user input)
[ ] File uploads: extension + magic byte validation, random filenames

Secrets & Crypto
[ ] No secrets in code or appsettings.json committed to git
[ ] Secrets in Key Vault / Secrets Manager in production
[ ] Passwords hashed with BCrypt/Argon2/PBKDF2
[ ] Sensitive data encrypted at rest and in transit
[ ] No MD5/SHA1 for security-sensitive hashing

CSRF
[ ] Antiforgery tokens on all state-changing forms
[ ] SameSite=Strict on auth cookies
[ ] CORS policy is restrictive (not AllowAnyOrigin in prod)

Logging & Monitoring
[ ] Security events logged (login failures, privilege escalation)
[ ] No sensitive data in logs (passwords, tokens, PII)
[ ] Centralized logging with alerts configured
[ ] Failed login rate limiting implemented

Dependencies
[ ] dotnet list package --vulnerable run in CI
[ ] Dependabot/NuGet Audit configured
[ ] No debug endpoints in production (Swagger behind auth)
[ ] Stack traces not exposed to users (UseExceptionHandler in prod)
```