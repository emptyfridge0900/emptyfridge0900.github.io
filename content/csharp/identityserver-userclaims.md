+++
title = "Understanding IdentityServer UserClaims: The Difference Between User, Identity Resource, and API Resource"
date = 2026-07-06

[taxonomies]
categories = ["post"]
tags = ["C#", "ASP.NET Core", "IdentityServer", "OAuth 2.0", "OpenID Connect", "Claims"]
+++

When you first look at the IdentityServer management UI, the name `UserClaims` appears in several places.

- UserClaims on Users/Roles users
- UserClaims on Identity Resources
- UserClaims on API Resources
- UserClaims on API Scopes, depending on the version

They share the same name, but they are not fields that store the same kind of data. If you miss this distinction, it is easy to think that an Identity Resource owns a user's claims, or that the User itself is also a Resource.

The core idea can be summarized in one sentence.

> User UserClaims are real type/value data, while Resource UserClaims are selection lists for claim types needed in tokens.

## Claims consist of a type and a value

A claim is one fact about a subject.

```text
type = department
value = Engineering
```

```text
type = app_role
value = platform_admin
```

`department` and `app_role` are claim types. `Engineering` and `platform_admin` are claim values for a particular user.

In .NET, they are represented like this.

```cs
new Claim("department", "Engineering");
new Claim("app_role", "platform_admin");
```

A claim type is a string. Inside a token, `role` is also just a string type like any other claim. Whether a claim is interpreted as a name or role is decided not by the claim itself, but by IdentityServer's issuance process and ASP.NET Core's consuming configuration.

## Claim, Scope, and Resource answer different questions

Claim, Scope, and Resource all appear during token issuance, but they are not concepts at the same layer.

| Concept | Question it answers | Examples |
|---|---|---|
| Claim | What facts are known about this user, client, or token? | `sub = alice-id`, `email = alice@example.com`, `app_role = platform_admin` |
| Scope | What access range did the client request and receive permission for? | `openid`, `profile`, `omniapi.read` |
| Resource | What is being protected? | User identity data, the `omniapi` API |

In one request, the three concepts connect in this order.

```text
Client
  │
  │ scope=openid profile omniapi.read
  ▼
IdentityServer
  ├── Which Identity Resource was requested?
  ├── Which API Scope was requested?
  ├── Which API Resource does that API Scope belong to?
  └── Which claim types does each Resource/Scope require?
        │
        ▼
Profile Service loads claim values for the current User
        │
        ├── identity token / userinfo
        └── access token
```

### Claims are data used for decisions

A claim is an assertion about a subject or client.

```text
sub      = alice-id
email    = alice@example.com
app_role = platform_admin
```

A claim is not an access request by itself. Even if the token contains `app_role = platform_admin`, it is not used for authorization unless the API defines an authorization rule that checks that claim.

Conversely, claims are not limited to permissions. Simple identity information such as `name`, `locale`, and `birthdate` are also claims. Claims such as `iss`, `aud`, and `exp` describe and validate the token itself.

### Scope is the access range requested by the client

In OAuth, a scope is a space-delimited, case-sensitive string.

```text
scope=omniapi.read omniapi.write
```

The authorization server and API define the concrete meaning of each scope name.

```text
omniapi.read  = read access to Omni API
omniapi.write = write access to Omni API
```

Just because a client requests a scope does not mean it is always granted. Only the scopes actually approved through the client's AllowedScopes, user consent, and server policy are reflected in the access token.

The word `scope` can appear in two places.

```text
Parameter in the authorization request
  scope=omniapi.read

Claim in the issued access token
  scope=omniapi.read
```

The first one is the client's request. The second one is the result that the authorization server sends to the API, meaning the range that was actually granted. Scope is therefore a protocol concept, and its result can be serialized inside the token as a `scope` claim.

### Resource is the thing being protected

A Resource is the protected target the client wants to access.

IdentityServer models Resources in two categories.

```text
Resource
  ├── Identity Resource: identity information about the user
  └── API Resource: an API or feature to call
```

An example of an Identity Resource is `profile`. The protected target here is user information such as `name`, `email`, and `locale`.

An example of an API Resource is `omniapi`. The protected target here is the Omni API's endpoints and features. The API Resource name may be used to build the `aud` audience in an access token.

You can think about the difference between Resource and Scope like this.

```text
Resource = Where are you accessing?
Scope    = How much are you allowed to do there?
Claim    = What facts are used for decisions and delivery?
```

For example, `omniapi` is a Resource, while `omniapi.read` and `omniapi.write` are different Scopes that can be allowed within that Resource. `app_role = platform_admin` is a Claim that the API can use for additional authorization decisions.

## How Identity Resources and API Resources process a request

Consider this configuration.

```text
Identity Resources
  openid  → [sub]
  profile → [name, email, locale]

API Resource
  omniapi
    Scopes → [omniapi.read, omniapi.write]
    UserClaims → [app_role]

API Scope
  omniapi.read  → []
  omniapi.write → [approval_level]
```

The client requests this.

```text
scope=openid profile omniapi.read
```

IdentityServer interprets the requested names like this.

| Requested scope | Configuration found | Result |
|---|---|---|
| `openid` | Identity Resource `openid` | Requests the `sub` type |
| `profile` | Identity Resource `profile` | Requests the `name`, `email`, and `locale` types |
| `omniapi.read` | API Scope `omniapi.read` | Determines the granted scope range |
| Resource connected to `omniapi.read` | API Resource `omniapi` | Determines the access token audience and common `app_role` type |

Simplified, the result looks like this.

```text
identity information side
  openid + profile
    → sub, name, email, locale
    → identity token / userinfo processing

API access side
  omniapi.read
    → aud = omniapi
    → scope = omniapi.read
    → request app_role Claim
    → access token processing
```

Identity scopes and API scopes can appear together in the same `scope` parameter, but their roles are different. Identity scopes select the set of user information to disclose. API scopes select delegated access ranges for an API.

## Why API Resources have Scopes but Identity Resources do not

It is not that Identity Resources have no scope. More precisely, **an Identity Resource itself is requested like one named scope.**

```cs
new IdentityResource(
    name: "profile",
    userClaims: new[] { "name", "email", "locale" });
```

In this configuration, the `Name`, `profile`, is the scope value requested by the client.

```text
scope=profile
```

The Identity Resource relationship is basically a one-to-one mapping.

```text
Identity Resource name
        │
        │ requested as a scope with the same name
        ▼
Claim type bundle

profile → [name, email, locale]
email   → [email, email_verified]
openid  → [sub]
```

Therefore, there is no need for another child scope collection under an Identity Resource. The Identity Resource named `profile` already acts as the configuration object for the `profile` scope.

In contrast, an API Resource represents the API itself or the audience. A single API can expose several access ranges.

```text
API Resource: omniapi
  ├── omniapi.read
  ├── omniapi.write
  ├── omniapi.export
  └── omniapi.admin
```

That is why API Resources have a `Scopes` collection that connects several API Scope names.

```cs
new ApiResource("omniapi", "Omni API")
{
    Scopes =
    {
        "omniapi.read",
        "omniapi.write",
        "omniapi.export"
    }
};
```

Each API Scope is defined separately and can be requested independently by a client.

```cs
new ApiScope("omniapi.read", "Read Omni data");
new ApiScope("omniapi.write", "Write Omni data");
new ApiScope("omniapi.export", "Export Omni data");
```

The structure is different because the target is modeled differently.

```text
Identity Resource
  one scope name = one identity Claim bundle

API Resource
  one API/audience = multiple independent API Scopes
```

### Difference between API Resource UserClaims and API Scope UserClaims

Both API Resources and API Scopes can have UserClaims. The difference is the range where they apply.

```text
API Resource omniapi
  UserClaims = [app_role]

API Scope omniapi.write
  UserClaims = [approval_level]
```

- `app_role`: common user information for the `omniapi` Resource
- `approval_level`: information required when the `omniapi.write` Scope is requested

If the client requests only `omniapi.read`, `app_role` is requested but `approval_level` is not. If the client requests `omniapi.write`, both the common Resource claim and the write Scope-specific claim may be requested.

So the place to put a UserClaim is decided by these questions.

```text
Is this Claim needed across the whole API?
  → API Resource.UserClaims

Is it needed only for a specific access range?
  → API Scope.UserClaims
```

## The three UserClaims locations have different roles

You need to distinguish the same name, `UserClaims`, by location.

| Location | What it stores | Role |
|---|---|---|
| User → UserClaims | Type and value | Real data belonging to a specific user |
| Identity Resource → UserClaims | Type names | List of user information requested by an identity scope |
| API Resource / API Scope → UserClaims | Type names | List of user information requested for an access token |

The first one is **data**. The other two are **selectors** or a **packing list**.

## User UserClaims are real per-user data

Suppose Alice has these claims stored.

```text
User: Alice

department = Engineering
app_role   = platform_admin
locale     = ko-KR
```

Bob can have different values.

```text
User: Bob

department = Support
app_role   = platform_member
locale     = en-US
```

These values are per-user data. If ASP.NET Core Identity is used as the store, raw user claims are usually stored in a separate table such as `AspNetUserClaims`. You can understand this as "claims attached to the User entity," but it does not necessarily mean they are columns on the User table.

It is also possible to have multiple values for the same type.

```text
role = Admin
role = Auditor
```

The Resource configuration lists `role` only once because it selects the type `role`, not one specific value. If the Profile Service issues both values, the token may represent them as an array or as multiple claims.

```json
{
  "role": ["Admin", "Auditor"]
}
```

You should avoid storing the exact same type/value pair more than once because it has no practical meaning.

## Identity Resource UserClaims are identity claim type lists

An Identity Resource does not represent one user. It is a **named bundle of identity information** that a client can request through the OIDC `scope` parameter.

For example, this Identity Resource groups the `name`, `email`, and `locale` types.

```cs
new IdentityResource(
    name: "profile",
    userClaims: new[] { "name", "email", "locale" },
    displayName: "User profile");
```

The client requests the `profile` scope like this.

```text
scope=openid profile
```

Here, the Identity Resource UserClaims do not contain values.

```text
name
email
locale
```

The list means this.

> Ask the Profile Service for the current logged-in user's `name`, `email`, and `locale` values.

If Alice has `locale = ko-KR`, Alice's value is issued. If Bob has `locale = en-US`, Bob's value is issued even though the same Identity Resource was requested. The Resource does not store `ko-KR` or `en-US`.

Claims selected by an Identity Resource are used for identity token or userinfo processing. Exactly which response includes them depends on the flow and IdentityServer configuration, but their purpose is different from API Scopes that represent API-call permissions.

## API Resource UserClaims are claim type lists for access tokens

An API Resource represents a logical API to protect. It is usually related to the access token audience and can group several API Scopes.

```cs
var omniApi = new ApiResource("omniapi", "Omni API")
{
    Scopes = { "omniapi.read", "omniapi.write" },
    UserClaims = { "app_role" }
};
```

This configuration does not mean the `app_role` value is stored inside the API Resource.

> When creating an access token for `omniapi`, request the current user's `app_role` Claim.

If Alice's user data contains this value,

```text
app_role = platform_admin
```

the Profile Service can read that value and put it into the access token.

```json
{
  "sub": "alice-id",
  "aud": "omniapi",
  "scope": "omniapi.read",
  "app_role": "platform_admin"
}
```

UserClaims can also be configured on API Scopes. The difference is the unit of application.

- API Resource UserClaims: claims commonly needed across scopes belonging to the Resource
- API Scope UserClaims: claims needed only when a specific scope is requested

For example, if `approval_level` is required only for `omniapi.write`, it can live on the API Scope.

```cs
new ApiScope(
    name: "omniapi.write",
    displayName: "Write access",
    userClaims: new[] { "approval_level" });
```

## Profile Service connects data with the selection lists

Resource configuration does not directly read claim values and copy them into a token. IdentityServer calls the Profile Service when claims are needed.

The important inputs received by the Profile Service include:

- which client made the request
- who the current user is
- whether the request is for an identity token, access token, or userinfo
- which claim types were requested

Simplified, the flow looks like this.

```text
Client requests scopes/resources
        ↓
IdentityServer collects claim types from Resource configuration
        ↓
Passes RequestedClaimTypes to the Profile Service
        ↓
Profile Service loads or computes actual values for the current User
        ↓
Issues allowed Claims into the token/userinfo response
```

Conceptually, the code looks like this.

```cs
public async Task GetProfileDataAsync(ProfileDataRequestContext context)
{
    var userClaims = await LoadClaimsForCurrentUser(context.Subject);

    context.IssuedClaims.AddRange(
        userClaims.Where(claim =>
            context.RequestedClaimTypes.Contains(claim.Type)));
}
```

A real implementation also needs to handle whether the user is active, client-specific disclosure rules, consent, sensitive data, duplicate claims, and similar concerns. The important point is that the Profile Service is the connection point.

Claim values do not have to come from UserClaims in the Users/Roles screen. The Profile Service can use many sources.

- raw user claims
- managed role assignment
- a separate application database
- LDAP or an external identity provider
- values computed at request time

Resource UserClaims do not specify the source. They specify only the required **types**.

## User is not a Resource

In IdentityServer terminology, User and Resource play different roles.

- User: the person who logs in through a client and accesses a Resource, meaning the token subject
- Client: the application that requests tokens
- Identity Resource: a requestable bundle of user identity information
- API Resource: an API or feature to protect

In OAuth terminology, the User is sometimes called the resource owner. But resource owner and the IdentityServer configuration object named Resource do not mean the same thing.

```text
User ──uses──▶ Client ──requests token──▶ IdentityServer
  │                                           │
  └────────── accesses Resource ◀─────────────┘
```

Because Identity Resources bundle user claims, the User itself can look like a Resource. But an Identity Resource does not represent the User object. It represents **part of the user's information** defined so clients can request it.

For example, `profile` is not the user Alice.

```text
profile = a requestable schema: [name, email, locale]
Alice   = the subject that provides actual values for that schema
```

## Which parts are standards, and which parts are IdentityServer implementation?

IdentityServer did not invent the entire concept of claims. It provides its own configuration and extension model on top of protocols defined by OAuth 2.0, OpenID Connect, and JWT.

| Structure | Standard? | Defined by |
|---|---|---|
| JWT claim set and `iss`, `sub`, `aud`, `exp` | Standard | RFC 7519 |
| ID token, UserInfo, `openid`, `profile`, `email` scope | Standard | OpenID Connect |
| `IdentityResource`, `ApiResource`, `UserClaims` collection | Product implementation | Duende IdentityServer |
| `ProfileService` and store interfaces | Product implementation | Duende IdentityServer |
| `Claim`, `ClaimsIdentity`, `ClaimsPrincipal` | Framework implementation | .NET |
| `AspNetUserClaims`, `AspNetUserRoles` table | Framework implementation | ASP.NET Core Identity |

Standards mainly define the messages exchanged between systems and their meanings.

```text
Identity Provider
    │
    │ standardized protocol/message
    │ ID token, UserInfo, JWT claim set
    ▼
Client / API
```

For example, JWT defines how to represent a JSON claim set and the registered claim names.

```json
{
  "iss": "https://identity.example.com",
  "sub": "alice-id",
  "aud": "omniapi",
  "exp": 1783378800,
  "app_role": "platform_admin"
}
```

`iss`, `sub`, `aud`, and `exp` are JWT registered claims. `app_role` is a private claim defined by the application. JWT allows registered, public, and private claim names.

OpenID Connect defines standard claims such as `sub`, `name`, `email`, and `picture`, and scopes such as `openid`, `profile`, and `email` that request them. But it does not define a C# class named `IdentityResource` or a UserClaims field in a management UI.

IdentityServer's `IdentityResource` is a configuration abstraction for implementing this standard behavior.

```text
OIDC standard concept
  profile scope → request Claims such as name, family_name, picture

IdentityServer implementation
  IdentityResource("profile")
      UserClaims = [name, family_name, picture]
```

API Resources and Profile Service are similar. Protocol requirements such as protected APIs, audiences, scopes, and user claim issuance come from standards, but configuring them through an `ApiResource` class and an `IProfileService` interface is IdentityServer's design.

Also, OAuth access tokens are not always JWTs. An authorization server can issue opaque access tokens. Only when JWT is used can the API directly see the JSON claim set.

## Is the claim storage structure standardized?

OAuth, OpenID Connect, and JWT do not prescribe the internal database schema of an IdP. Any of the following can be used as long as the token follows the protocol rules and remains interoperable.

```text
Inside the Identity Provider
  ├── relational database
  ├── ASP.NET Core Identity tables
  ├── LDAP / Active Directory
  ├── external identity provider
  ├── separate user-management API
  └── claims computed at request time
```

Therefore, this table structure is not a standard.

```text
AspNetUsers
AspNetUserClaims
AspNetRoles
AspNetUserRoles
```

Another IdP may use a completely different schema. IdentityServer also provides store interfaces for configuration, resources, persisted grants, and similar data, and you can choose an in-memory store or a custom database implementation. The user store is not fixed to ASP.NET Core Identity either.

Standards do not force an internal store because claim sources vary. `department` can come from an HR database, `role` from directory groups, `email` from an external IdP, and a request-time `risk_level` from a separate service. The Profile Service is IdentityServer's connection point that projects these sources into token claims.

## What does SCIM standardize?

SCIM, or System for Cross-domain Identity Management, is a separate standard for provisioning users and groups between systems.

SCIM defines:

- JSON resource schemas for Users and Groups
- attributes such as `userName`, `name`, `emails`, `groups`, `roles`, and `entitlements`
- schema extensions and namespaces
- an HTTP protocol for create, read, update, and delete operations

A SCIM User looks like this.

```json
{
  "schemas": [
    "urn:ietf:params:scim:schemas:core:2.0:User"
  ],
  "userName": "alice@example.com",
  "name": {
    "givenName": "Alice"
  },
  "roles": [
    { "value": "Admin" }
  ]
}
```

SCIM allows several systems to provision User and Group data in the same format. But it still does not decide:

- how the physical database tables should be designed
- which token claim a SCIM attribute should become
- what actual permission the `Admin` role grants
- which IdentityServer Resource should request that Claim

In other words, SCIM standardizes the **user-management API and exchange schema**, but not the IdP's physical storage structure or token issuance policy.

The overall boundary looks like this.

```text
SCIM
  standardizes User/Group provisioning and exchange schema
        ↓
IdP internal store
  designed freely by the product or organization
        ↓
IdentityServer configuration + Profile Service
  projects stored information into token Claims
        ↓
OAuth / OpenID Connect / JWT
  standardizes externally delivered protocols and messages
        ↓
.NET Claim / ClaimsIdentity / ClaimsPrincipal
  converted into runtime objects inside the API process
```

## Is `role` a reserved Claim type?

`role` is not a special reserved data type at the token-format level. It is a string claim type.

```cs
new Claim("role", "Admin");
```

In real systems, however, it can be treated specially by convention.

1. The identity provider's role-management feature may project role assignments into `role` Claims.
2. If a Resource requests the `role` type, those values may be included in the token.
3. If ASP.NET Core configures `role` as the `RoleClaimType`, `IsInRole()` interprets that Claim as a role.

```cs
options.TokenValidationParameters.RoleClaimType = "role";
```

```cs
User.IsInRole("Admin");
```

Conceptually, the code above checks whether the current `ClaimsIdentity` has a Claim with `type = role` and `value = Admin`.

So `role` is technically an arbitrary string, but if your system adopts it as the role namespace, it should be treated operationally as a name with ownership.

## Raw user claim and managed role assignment

There can be two ways to give role information to a user.

### Raw user claim

```text
UserClaims
  type  = role
  value = Admin
```

### Managed role assignment

```text
Roles
  Admin

UserRoles
  Alice → Admin
```

The storage structures are different, but if the Profile Service projects both the same way, the token result can be the same.

```json
{
  "role": "Admin"
}
```

A managed role gives structure such as role definitions, user assignment, group inheritance, and management UI. A raw user claim is simple and flexible, but typos and invalid values are harder to control.

If both methods are used together, duplicate and conflict rules must be clear.

```text
Role assignment → role = Admin
Raw user claim  → role = Support
```

If both are issued, the API sees the user as having both the `Admin` and `Support` roles. From the token alone, the API cannot tell which storage source each value came from.

## In a shared IdP, separate claim types with namespaces

If several products share one IdP, a generic `role` is easy to collide.

```text
role = Admin
```

From that value alone, you cannot know which product's Admin role it is. If several products use the same type and value, another product's role can be interpreted as the wrong permission.

Product-specific types create clearer boundaries.

```text
app_role     = platform_admin
billing_role = auditor
support_role = agent
```

Omni API can configure only its own type as the role type.

```cs
options.TokenValidationParameters.RoleClaimType = "app_role";
```

Then this check looks only at `app_role`.

```cs
User.IsInRole("platform_admin");
```

Values such as `platform_admin` and `platform_member` are safer than context-poor names like `Admin` or `Manager`. Claim value comparison usually uses exact string matching, so casing and spelling also need to be managed as part of the contract.

## Revisit the full example

### 1. Store actual values on the User

```text
Alice
  sub      = alice-id
  name     = Alice
  email    = alice@example.com
  app_role = platform_admin
```

### 2. Identity Resource selects identity claim types

```text
Identity Resource: profile
UserClaims: name, email
```

### 3. API Resource selects access token claim types

```text
API Resource: omniapi
Scopes: omniapi.read, omniapi.write
UserClaims: app_role
```

### 4. Client requests scopes

```text
scope=openid profile omniapi.read
```

### 5. Profile Service finds Alice's values

```text
Requested identity types: name, email
Requested API type: app_role

Alice's values:
  name = Alice
  email = alice@example.com
  app_role = platform_admin
```

### 6. The API sees only the access token

```json
{
  "sub": "alice-id",
  "aud": "omniapi",
  "scope": "omniapi.read",
  "app_role": "platform_admin"
}
```

By the time the request reaches the API, the Identity Resource, API Resource, and Users/Roles screen structure are gone. The JWT bearer handler validates the token and creates a `ClaimsIdentity` and `ClaimsPrincipal` from the claim list. The API consumes only the final claims.

## Common misunderstandings

### Thinking Resource UserClaims store values

Resources store only type names. The actual value is provided by the current User and the Profile Service.

### Thinking claims stored in UserClaims automatically appear in every token

Storage and issuance are separate. The Profile Service and filtering policy for the requested scopes/resources must issue that Claim.

### Thinking Identity Resources and API Resources shape the same token

Both select user claim types, but they have different purposes. Identity Resources model disclosure of identity information. API Resources and API Scopes model information needed in API access tokens.

### Thinking the name `role` grants permission by itself

A Claim is data. It affects authorization only when the API configures that type as the `RoleClaimType` or checks it in an authorization policy.

### Thinking User is an Identity Resource

User is the subject/resource owner. An Identity Resource is a requestable bundle of information about that User.

## Design checklist

- Have you clearly separated claim type and value?
- Have you separated actual User data from Resource type-selection lists?
- Have you decided whether each Claim is needed in identity token/userinfo or in access token?
- Are the claim value sources and Profile Service projection rules clear?
- If the same type has multiple values, does the API handle them correctly?
- Are raw user claims and managed role assignments avoiding duplication?
- In a shared IdP, are product-specific claim type namespaces being used?
- Does the API's `RoleClaimType` exactly match the issued claim type?
- Have you decoded the token and verified `aud`, `scope`, and claim type/value in practice?

## Summary

IdentityServer repeats the name `UserClaims` in several screens, but the meaning splits into two categories.

```text
User UserClaims
  = actual type/value data belonging to the current user

Resource UserClaims
  = list of claim type names to request from the Profile Service
```

An Identity Resource is not the user itself; it is a requestable bundle of identity information. An API Resource is the protected API. The Profile Service connects the types selected by Resources with the current User's actual values and issues them into tokens.

Once this structure is clear, `role` also becomes easier to understand. `role` is a string Claim type, but role projection by the IdP and ASP.NET Core's `RoleClaimType` configuration make that Claim participate in role authorization. In a shared IdP, using a namespaced type such as `app_role` is safer.

## Ref

- Duende IdentityServer terminology: <https://docs.duendesoftware.com/identityserver/overview/terminology/>
- Duende IdentityServer claims: <https://docs.duendesoftware.com/identityserver/fundamentals/claims/>
- Duende IdentityServer resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/>
- Duende IdentityServer identity resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/identity/>
- Duende IdentityServer Identity Resource model: <https://docs.duendesoftware.com/identityserver/reference/v8/models/identity-resource/>
- Duende IdentityServer API resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/api-resources/>
- Duende IdentityServer API scopes: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/api-scopes/>
- Duende IdentityServer Profile Service: <https://docs.duendesoftware.com/identityserver/reference/v8/services/profile-service/>
- Duende IdentityServer stores: <https://docs.duendesoftware.com/identityserver/reference/v7/stores/>
- ASP.NET Core role-based authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles>
- `ClaimsIdentity.RoleClaimType`: <https://learn.microsoft.com/en-us/dotnet/api/system.security.claims.claimsidentity.roleclaimtype>
- `TokenValidationParameters.RoleClaimType`: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.tokens.tokenvalidationparameters.roleclaimtype>
- `ClaimsPrincipal.IsInRole`: <https://learn.microsoft.com/en-us/dotnet/api/system.security.claims.claimsprincipal.isinrole>
- OAuth 2.0 Authorization Framework, RFC 6749: <https://www.rfc-editor.org/rfc/rfc6749>
- JSON Web Token, RFC 7519: <https://www.rfc-editor.org/rfc/rfc7519>
- OpenID Connect Core 1.0: <https://openid.net/specs/openid-connect-core-1_0.html>
- SCIM Core Schema, RFC 7643: <https://www.rfc-editor.org/rfc/rfc7643>
- SCIM Protocol, RFC 7644: <https://www.rfc-editor.org/rfc/rfc7644>
