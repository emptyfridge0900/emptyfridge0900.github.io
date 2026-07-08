+++
title = "Understanding ASP.NET Core Identity, OpenIddict, and Claims Authentication"
date = 2026-07-04

[taxonomies]
categories = ["post"]
tags = ["C#", "ASP.NET Core", "Identity", "OpenIddict", "OAuth 2.0", "OpenID Connect"]
+++

When studying authentication in ASP.NET Core, Identity, OpenIddict, JWT, Claim, Role, and Policy all appear at once. They are all related to authentication and authorization, but they do not solve the same problem. The overall structure becomes especially hard to understand if ASP.NET Core Identity and OpenIddict are treated as competitors, or if JWT itself is considered the thing that authenticates users.

This article organizes each component's responsibility and how they connect when handling a request.

## Start by separating authentication and authorization

First, authentication and authorization need to be separated.

- Authentication: confirms who sent the request.
- Authorization: decides whether the authenticated subject can perform a specific action.

Checking a password during login is authentication. Checking whether the logged-in user can access an admin page is authorization.

## Roles of ASP.NET Core Identity and OpenIddict

### ASP.NET Core Identity

ASP.NET Core Identity manages application user accounts and login features.

- Storing users and password hashes
- Registration and login
- Managing roles and user claims
- Email confirmation and password reset
- Two-factor authentication
- Creating authentication cookies
- Login through external providers such as Google or Microsoft

In short, Identity's core question is:

> Who is this user, and how does the user prove their identity?

Even though Identity can use external OAuth/OIDC providers, Identity itself does not become an OAuth 2.0 or OpenID Connect server. In that case, the application acts as a **client** of an external provider such as Google.

### OpenIddict

OpenIddict is a framework for implementing OAuth 2.0/OpenID Connect protocols. With its server stack, an application can become an authorization server or OpenID Provider.

- Handling protocol endpoints such as authorization, token, and logout
- Managing OAuth client applications
- Managing scopes and authorizations
- Issuing access tokens, refresh tokens, and ID tokens
- Token validation, introspection, and revocation
- Handling authorization code, client credentials, device flow, and similar flows

OpenIddict's core question is:

> Through what process does a client receive a token, and which APIs can that token access?

OpenIddict is not a complete user-management product. Login pages, registration, password recovery, and user management UI need to be implemented through another system such as Identity.

## They can be used together, but they do not depend on each other

Identity and OpenIddict are often used together like this.

```text
User credentials
    ↓
ASP.NET Core Identity
    ↓ creates ClaimsPrincipal for the authenticated user
OpenIddict
    ↓ issues OAuth/OIDC token
Client application
    ↓ requests with access token
Protected API
```

But OpenIddict does not internally use Identity. The `ClaimsPrincipal` passed to OpenIddict can be created from many places.

- ASP.NET Core Identity
- A separate user table
- LDAP or Active Directory
- External authentication provider
- Client credentials flow without a user

Identity is only one convenient way to authenticate users and create a `ClaimsPrincipal`.

## Relationship between JWT, Claim, ClaimsIdentity, ClaimsPrincipal, and User

A typical API request using JWT authentication is processed in this order.

```text
JWT
  → token handler validates signature, issuer, audience, expiration, etc.
  → payload values are converted into Claims
  → ClaimsIdentity is created with those Claims
  → one or more ClaimsIdentity instances are wrapped in ClaimsPrincipal
  → assigned to HttpContext.User
```

### JWT

JWT is a token **format** for carrying information. The payload can contain claims such as:

```json
{
  "sub": "user-123",
  "name": "Alice",
  "role": "Admin",
  "tenant_id": "tenant-7",
  "iss": "https://identity.example.com",
  "aud": "orders-api",
  "exp": 1783123200
}
```

A JWT is not the same object as a `ClaimsPrincipal`, and not every `ClaimsPrincipal` is created from a JWT. Cookies, client certificates, external login, and custom authentication handlers can also create a `ClaimsPrincipal`.

### Claim

A `Claim` is one type/value item about an authenticated subject.

```cs
new Claim("sub", "user-123");
new Claim("department", "Engineering");
new Claim("role", "Admin");
```

A claim is just data. Having a claim does not automatically grant access. Authorization rules decide how to interpret that claim.

### ClaimsIdentity

`ClaimsIdentity` contains a set of claims and information about the authentication method.

```cs
var identity = new ClaimsIdentity(
    claims,
    authenticationType: "Bearer",
    nameType: "name",
    roleType: "role");
```

Here, `nameType` and `roleType` specify which claim should be interpreted as the name and role.

### ClaimsPrincipal

`ClaimsPrincipal` contains one or more `ClaimsIdentity` instances. A single subject can have identities from multiple authentication methods.

```cs
var principal = new ClaimsPrincipal(identity);
```

### HttpContext.User

In ASP.NET Core, `HttpContext.User`, or `User` in a controller, is not a separate user entity. It is a property pointing to the current request's `ClaimsPrincipal`.

Therefore, the following code does not read the token or cookie directly. It queries the `ClaimsPrincipal` created by the authentication handler.

```cs
var subject = User.FindFirst("sub")?.Value;
var authenticated = User.Identity?.IsAuthenticated == true;
var isAdmin = User.IsInRole("Admin");
```

Identity's `ApplicationUser` and `HttpContext.User` should also be separated. `ApplicationUser` is a user entity stored in the database, while `HttpContext.User` is the security context included in the current request.

## Why is role-based authorization claim-based?

At ASP.NET Core runtime, a role is represented as a claim with the configured role claim type.

```cs
[Authorize(Roles = "Admin")]
public IActionResult AdminOnly() => Ok();
```

Conceptually, this check asks:

```text
Among the authenticated ClaimsIdentity instances in the current ClaimsPrincipal,
is there a Claim whose type matches RoleClaimType and whose value is "Admin"?
```

The default `RoleClaimType` is `ClaimTypes.Role`, but that is not the only possible value. It can also be configured to use the JWT-native name `role`.

```cs
options.TokenValidationParameters = new TokenValidationParameters
{
    RoleClaimType = "role",
    NameClaimType = "name"
};
```

So instead of saying "a role is always a `ClaimTypes.Role` claim," this is more accurate:

> ASP.NET Core interprets claims whose type matches `ClaimsIdentity.RoleClaimType` as roles.

## Claim-based authorization and policy-based authorization

Policy is the general execution model for ASP.NET Core authorization. A policy is composed of one or more requirements, and authorization handlers evaluate those requirements.

```cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("EngineeringOnly", policy =>
        policy.RequireClaim("department", "Engineering"));
});
```

```cs
[Authorize(Policy = "EngineeringOnly")]
public IActionResult EngineeringDashboard() => Ok();
```

Both role checks and claim checks run as requirements in the policy engine.

```text
Policy-based authorization
  ├── Role requirement
  ├── Claim requirement
  ├── Authentication requirement
  └── Custom requirement + handler
```

So instead of equating "claim-based authorization" with "policy-based authorization," it is more accurate to understand it this way:

> Policy is the general framework for executing authorization rules, and role and claim checks are representative requirements provided by that framework.

Complex rules are hard to express with a single claim check. For example, a rule like "is the current user in this tenant and either the owner of this order or a support agent?" is better handled with resource-based authorization or a custom requirement.

```cs
public sealed class OrderAccessRequirement
    : IAuthorizationRequirement;

public sealed class OrderAccessHandler
    : AuthorizationHandler<OrderAccessRequirement, Order>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OrderAccessRequirement requirement,
        Order order)
    {
        var userId = context.User.FindFirst("sub")?.Value;
        var tenantId = context.User.FindFirst("tenant_id")?.Value;

        if (order.UserId == userId && order.TenantId == tenantId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

## Why ClaimTypes and JWT/OIDC claim names differ

.NET's `System.Security.Claims.ClaimTypes` provides well-known claim types as string constants.

```cs
ClaimTypes.Name
// http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name

ClaimTypes.NameIdentifier
// http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier

ClaimTypes.Role
// http://schemas.microsoft.com/ws/2008/06/identity/claims/role
```

### Are the long ClaimTypes strings XML namespace URIs?

Yes. The long strings in `ClaimTypes` use **XML namespace-style schema URIs** as claim type identifiers from the XML/SOAP era. WS-* technologies such as WIF(Windows Identity Foundation) and WS-Federation used URI namespaces so names defined by different standards and providers would not collide. `ClaimTypes` inherited that naming convention.

However, this does not mean they are literally `xmlns` declarations in an XML document. Values such as `ClaimTypes.Name` are strings that identify the .NET `Claim.Type`. It is more accurate to understand them as **schema URI identifiers** using the same URI namespacing style as XML namespaces.

```xml
<!-- XML namespaces also use URIs as namespace identifiers. -->
<claims xmlns="http://schemas.xmlsoap.org/ws/2005/05/identity/claims">
    <name>Alice</name>
</claims>
```

```cs
// In .NET, the full URI is the Claim.Type string.
var claim = new Claim(
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
    "Alice");

Debug.Assert(claim.Type == ClaimTypes.Name);
```

They start with `http://` and look like URLs, but they do not have to point to an actual network resource or be visitable in a browser. The purpose of the URI here is not location; it is to provide a globally distinguishable name. Microsoft documentation also describes these values as URIs for well-known claim types, and they should not be expected to behave as hyperlinks.

Because these values were created before JWT, they do not directly match the short JSON claim names used by OIDC/JWT.

| Meaning | .NET `ClaimTypes` URI | JWT/OIDC name |
|---|---|---|
| User identifier | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier` | `sub` |
| Display name | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` | `name` |
| Email | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` | `email` |
| Role | `http://schemas.microsoft.com/ws/2008/06/identity/claims/role` | `role` or `roles`, depending on implementation |

JWT and OpenID Connect instead use short names that fit JSON well.

### "Registered Claim Name" is more accurate than "reserved claim"

JWT claims are often called "reserved claims," but RFC 7519's official term is **Registered Claim Name**. "Reserved" can misleadingly imply that the name must be used, or that every JWT must include it.

RFC 7519 divides claim names into three categories.

| Category | Meaning | Example |
|---|---|---|
| Registered | Names registered in the IANA registry so multiple systems can use the same meaning | `iss`, `sub`, `aud`, `exp` |
| Public | Names publicly registered or using collision-resistant names to avoid conflicts | `https://example.com/claims/tenant_id` |
| Private | Names agreed on by issuer and consumer, with no global meaning or collision guarantee | `tenant_id`, `permission` |

The important point is that **RFC 7519 itself does not require registered claims in every JWT**. Whether a claim is required is decided by the higher-level protocol or application profile that uses JWT.

For example, `exp` is optional in a generic JWT, but required in an OpenID Connect ID token. The correct explanation is not "it is required because it is a standard JWT claim," but "the OIDC ID token rules require `exp` in that JWT."

### Basic Registered Claim Names in RFC 7519

The seven claims originally registered by RFC 7519 are:

| Claim | Name | Value shape | Meaning and validation point |
|---|---|---|---|
| `iss` | Issuer | case-sensitive string or URI | Identifies the token issuer. Must exactly match the trusted issuer. |
| `sub` | Subject | case-sensitive string or URI | The subject described by the token. Must be unique within the issuer. It can be a user ID, or in client credentials flow it can represent the client. |
| `aud` | Audience | string or string array | Identifies the token recipient. If the current API is not in the audience, reject the token. |
| `exp` | Expiration Time | NumericDate | The token must not be accepted after this time. Validation can allow limited clock skew. |
| `nbf` | Not Before | NumericDate | The token must not be accepted before this time. |
| `iat` | Issued At | NumericDate | The time the token was issued. Can be used for token age calculations, but does not by itself determine expiration. |
| `jti` | JWT ID | case-sensitive string | Unique token identifier. Can be used for replay prevention or revocation records, but does not automatically prevent replay unless stored and checked. |

`NumericDate` is a JSON number representing seconds since 1970-01-01 UTC. It is not an ISO 8601 string.

```json
{
  "iss": "https://identity.example.com",
  "sub": "user-123",
  "aud": ["orders-api", "billing-api"],
  "exp": 1783123200,
  "nbf": 1783119300,
  "iat": 1783119600,
  "jti": "01JXYZ8M6Y4V7Q9R2K3P"
}
```

These values are not just user profile information; they form the token's security context. In particular, if you validate only the signature and not `iss`, `aud`, and `exp`, you can accidentally accept a token that was validly signed but issued for a different API.

### The IANA registry does not stop at RFC 7519's seven claims

The IANA JSON Web Token Claims registry also includes names registered later by other standards. Examples include:

- OIDC user information: `name`, `email`, `email_verified`, `picture`
- Authentication information: `auth_time`, `nonce`, `acr`, `amr`, `azp`
- OAuth information: `client_id`, `scope`
- Authorization information: `groups`, `roles`, `entitlements`

Therefore, "registered claim" does not mean only the seven RFC 7519 claims. It means the full set of names currently registered in the registry, and each claim's meaning and requirements should be checked in the specification that registered it.

Also distinguish singular `role` and plural `roles`. RFC 9068's JWT access token profile recommends `groups`, `roles`, and `entitlements` for authorization information. Many .NET or identity provider implementations, however, conventionally use `role` or the Microsoft URI form `ClaimTypes.Role`. The actual claim name in the token must match ASP.NET Core's `RoleClaimType`.

### Claims in an OpenID Connect ID token

An ID token is not "an OAuth access token with a little user information added." It is a separate token that lets the client verify the result of user authentication at the authorization server, and it is always a JWT.

Core claims required or defined by OIDC Core include:

| Claim | Requirement | Meaning |
|---|---|---|
| `iss` | Required | OpenID Provider that issued the ID token |
| `sub` | Required | User identifier that is unique and never reassigned within the issuer |
| `aud` | Required | Client that receives this ID token. Usually includes `client_id`. |
| `exp` | Required | ID token expiration time |
| `iat` | Required | ID token issued-at time |
| `auth_time` | Required depending on request | Time the user actually authenticated |
| `nonce` | Required if sent in the authorization request | Connects request and response and prevents replay attacks. The client must compare it with the original value. |
| `acr` | Optional | Authentication Context Class. Can indicate authentication strength or policy. |
| `amr` | Optional | Authentication Methods References. Example: password, OTP |
| `azp` | Used in specific conditions | Authorized party for which the ID token was issued. Important especially when there are multiple audiences. |

`sub` should be treated as a stable internal identifier rather than a login name or email address. Email can change or be reused, but OIDC `sub` is designed to identify the user stably within the issuer. When linking an external user in a database, `(iss, sub)` is usually used as the key because different issuers can issue the same `sub` value.

OIDC Standard Claims describe user profile information.

| Area | Claims |
|---|---|
| Name | `name`, `given_name`, `family_name`, `middle_name`, `nickname`, `preferred_username` |
| Profile | `profile`, `picture`, `website` |
| Contact | `email`, `email_verified`, `phone_number`, `phone_number_verified` |
| Other | `gender`, `birthdate`, `zoneinfo`, `locale`, `address`, `updated_at` |

These claims are not automatically included in the ID token just because they are defined. Depending on requested scopes, the `claims` parameter, user consent, and provider policy, they may be provided in the ID token or UserInfo response.

Representative scope-to-claim relationships are:

- `openid`: makes the request an OIDC request. It does not mean the full user profile.
- `profile`: requests profile claims such as name, picture, locale, and `updated_at`.
- `email`: requests `email` and `email_verified`.
- `phone`: requests `phone_number` and `phone_number_verified`.
- `address`: requests `address`.

A scope is not a simple command meaning "put this claim in the token." It represents the access range requested by the client, and the actual result is affected by consent and provider policy.

### Do not confuse ID token claims and access token claims

The audience of an ID token is the **client application**, while the audience of an access token is usually the **resource server/API**.

```text
ID token
  issuer ──authentication result──> client application

Access token
  issuer ──API access right──> resource server
```

Therefore, an API must not accept an ID token as a substitute for an access token. Even if both tokens are JWTs and contain similar claims, their purpose and audience differ.

Also, an OAuth access token does not have to be a JWT. OpenIddict can also use opaque access tokens. If a JWT access token follows the RFC 9068 profile, these claims are required:

- `iss`
- `sub`
- `aud`
- `exp`
- `iat`
- `jti`
- `client_id`

If the authorization request included scope, RFC 9068 recommends including a `scope` claim in the access token. But clients should treat access tokens as opaque values and not depend on their internal structure. The consumer of the access token is the API. If a client reads token payloads to decide UI permissions or login state, it becomes fragile when the token format changes.

### How to name custom claims

For claims used only inside an internal system, short private names can be used.

```json
{
  "tenant_id": "tenant-7",
  "permission": ["orders.read", "orders.write"]
}
```

But if tokens are exchanged across multiple organizations or products, short names can collide with future standards or claims from other providers. In that case, a collision-resistant name under a namespace you control is safer.

```json
{
  "https://auth.example.com/claims/tenant_id": "tenant-7"
}
```

A URI-shaped claim name does not need to be an actually reachable URL. The important part is that it is a unique identifier under a namespace you control. Note that RFC 7519's `StringOrURI` rule is about the data type of some **claim values**, such as `iss` or `sub`, and does not apply to all claim names.

When designing custom claims, check these points:

1. Is there already a registered claim with the same meaning in the IANA registry?
2. Is the consumer of this claim the client or the API?
3. Is the value shape defined by contract: string, array, boolean, etc.?
4. If naming conflicts are possible, are you using a collision-resistant name?
5. Is this sensitive information that the API could query from the server instead of putting it in the token?
6. If the value changes, is it acceptable for already issued tokens to keep stale permissions until expiration?

If a JWT payload is an unencrypted JWS, anyone can base64url-decode and read it. The signature prevents tampering, but it does not provide confidentiality. Do not put passwords, secrets, or unnecessary personal information in claims. For authorization data, also consider that values can remain stale until the token expires.

## Inbound claim mapping

The JWT bearer handler can map short token claim names to .NET URI-style claim types. For example, `sub` can be converted to `ClaimTypes.NameIdentifier`.

This behavior can be disabled per scheme.

```cs
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.MapInboundClaims = false;
        options.TokenValidationParameters = new TokenValidationParameters
        {
            NameClaimType = "name",
            RoleClaimType = "role"
        };
    });
```

If mapping is disabled, you can use token-native names such as `User.FindFirst("sub")`. The important part is not whether mapping is on or off, but that claim names, `NameClaimType`, and `RoleClaimType` are used consistently throughout the application.

## Identity bearer tokens and OAuth tokens are different

Recent ASP.NET Core Identity API endpoints can issue bearer tokens for first-party clients. However, these tokens are Identity-specific proprietary tokens; they do not implement an OAuth 2.0/OIDC token server.

If you need the following features, use an OAuth/OIDC server framework such as OpenIddict.

- Registering multiple client applications
- Authorization code + PKCE
- Standard scopes and consent
- ID tokens and OIDC discovery
- Refresh token policies
- Token introspection and revocation
- Standards-based interoperability with external applications

For new applications with users, prefer authorization code flow with PKCE over password grant. Password grant is not recommended because the client directly receives the user's password. For server-to-server communication, client credentials flow is appropriate.

## Full structure

```text
                        Authentication

Credential / Cookie / JWT / Certificate
                    ↓
          Authentication Handler
                    ↓
              ClaimsIdentity
                    ↓
             ClaimsPrincipal
                    ↓
           HttpContext.User

                         Authorization

           HttpContext.User + Resource
                    ↓
          Authorization Policy
                    ↓
    Role / Claim / Custom Requirements
                    ↓
              Allow or Deny
```

Finally, each term in one sentence:

- Identity: manages user accounts and the login lifecycle.
- OpenIddict: handles OAuth 2.0/OIDC protocols and issues or validates standard tokens.
- JWT: a signed token format that can carry claims.
- Claim: type/value information about a subject.
- ClaimsIdentity: a set of claims from one authentication source.
- ClaimsPrincipal: the current security principal containing one or more ClaimsIdentity instances.
- `HttpContext.User`: the current request's ClaimsPrincipal.
- Role: a claim that ASP.NET Core interprets as a role according to RoleClaimType.
- Policy: an authorization rule that evaluates roles, claims, and custom requirements.

## Ref

- ASP.NET Core Identity overview: <https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity>
- Identity API authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity-api-authorization>
- Claims-based authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/claims>
- Policy-based authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/policies>
- Role-based authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles>
- `ClaimTypes`: <https://learn.microsoft.com/en-us/dotnet/api/system.security.claims.claimtypes>
- `JwtBearerOptions.MapInboundClaims`: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authentication.jwtbearer.jwtbeareroptions.mapinboundclaims>
- OpenIddict introduction: <https://documentation.openiddict.com/introduction>
- OpenIddict getting started: <https://documentation.openiddict.com/guides/getting-started/>
- Choosing an OAuth/OIDC flow with OpenIddict: <https://documentation.openiddict.com/guides/choosing-the-right-flow.html>
- JSON Web Token, RFC 7519: <https://www.rfc-editor.org/rfc/rfc7519>
- IANA JSON Web Token Claims registry: <https://www.iana.org/assignments/jwt/jwt.xhtml>
- JWT Profile for OAuth 2.0 Access Tokens, RFC 9068: <https://www.rfc-editor.org/rfc/rfc9068>
- OpenID Connect Core 1.0: <https://openid.net/specs/openid-connect-core-1_0.html>
