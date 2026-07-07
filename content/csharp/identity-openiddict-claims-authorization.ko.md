+++
title = "ASP.NET Core Identity와 OpenIddict, Claims 인증 구조 이해하기"
date = 2026-07-04

[taxonomies]
categories = ["post"]
tags = ["C#", "ASP.NET Core", "Identity", "OpenIddict", "OAuth 2.0", "OpenID Connect"]
+++

ASP.NET Core 인증을 공부하다 보면 Identity, OpenIddict, JWT, Claim, Role, Policy가 한꺼번에 등장한다. 모두 인증과 인가에 관련된 개념이지만 같은 문제를 해결하지는 않는다. 특히 ASP.NET Core Identity와 OpenIddict를 경쟁 관계로 생각하거나, JWT 자체가 사용자를 인증한다고 생각하면 전체 구조를 이해하기 어렵다.

이 글에서는 각 구성 요소의 책임과 요청을 처리할 때 이들이 어떻게 연결되는지 정리한다.

## 인증과 인가부터 구분하기

먼저 인증(authentication)과 인가(authorization)를 구분해야 한다.

- 인증: 요청을 보낸 주체가 누구인지 확인한다.
- 인가: 인증된 주체가 특정 작업을 수행할 수 있는지 판단한다.

로그인 과정에서 비밀번호를 확인하는 것은 인증이다. 로그인한 사용자가 관리자 페이지에 접근할 수 있는지 확인하는 것은 인가다.

## ASP.NET Core Identity와 OpenIddict의 역할

### ASP.NET Core Identity

ASP.NET Core Identity는 애플리케이션의 사용자 계정과 로그인 기능을 관리한다.

- 사용자와 비밀번호 해시 저장
- 회원 가입과 로그인
- 역할과 사용자 Claim 관리
- 이메일 확인과 비밀번호 초기화
- 2단계 인증
- 인증 Cookie 생성
- Google이나 Microsoft 같은 외부 공급자를 통한 로그인

즉, Identity의 핵심 질문은 다음과 같다.

> 이 사용자는 누구이며, 자신의 신원을 어떻게 증명하는가?

Identity가 외부 OAuth/OIDC 공급자를 이용할 수 있다고 해서 Identity 자체가 OAuth 2.0 또는 OpenID Connect 서버가 되는 것은 아니다. 이 경우 애플리케이션은 Google 같은 외부 공급자의 **클라이언트**로 동작한다.

### OpenIddict

OpenIddict는 OAuth 2.0/OpenID Connect 프로토콜을 구현하기 위한 프레임워크다. 서버 스택을 사용하면 애플리케이션을 authorization server 또는 OpenID Provider로 만들 수 있다.

- authorization, token, logout 같은 프로토콜 endpoint 처리
- OAuth client application 관리
- scope와 authorization 관리
- access token, refresh token, ID token 발급
- token validation, introspection, revocation
- authorization code, client credentials, device flow 등의 처리

OpenIddict의 핵심 질문은 다음과 같다.

> 클라이언트가 어떤 절차로 token을 받고, 그 token으로 어떤 API에 접근할 수 있는가?

OpenIddict는 완성된 사용자 관리 제품이 아니다. 로그인 화면, 회원 가입, 비밀번호 복구, 사용자 관리 UI는 Identity 같은 별도 시스템으로 구현해야 한다.

## 둘은 함께 쓸 수 있지만 서로 의존하지 않는다

Identity와 OpenIddict는 보통 다음과 같이 함께 사용한다.

```text
사용자 자격 증명
    ↓
ASP.NET Core Identity
    ↓ 인증된 사용자 정보를 ClaimsPrincipal로 생성
OpenIddict
    ↓ OAuth/OIDC token 발급
클라이언트 애플리케이션
    ↓ access token으로 요청
보호된 API
```

하지만 OpenIddict가 내부적으로 Identity를 사용하는 것은 아니다. OpenIddict에 전달할 `ClaimsPrincipal`은 여러 곳에서 만들 수 있다.

- ASP.NET Core Identity
- 별도의 사용자 테이블
- LDAP 또는 Active Directory
- 외부 인증 공급자
- 사용자 없이 동작하는 client credentials flow

Identity는 사용자를 인증하고 `ClaimsPrincipal`을 만드는 편리한 방법 중 하나일 뿐이다.

## JWT, Claim, ClaimsIdentity, ClaimsPrincipal, User의 관계

JWT 인증을 사용하는 일반적인 API 요청은 다음 순서로 처리된다.

```text
JWT
  → token handler가 서명, issuer, audience, 만료 시간 등을 검증
  → payload의 값들을 Claim으로 변환
  → Claim들을 가진 ClaimsIdentity 생성
  → 하나 이상의 ClaimsIdentity를 ClaimsPrincipal로 감쌈
  → HttpContext.User에 할당
```

### JWT

JWT는 정보를 전달하는 token **형식**이다. payload에는 다음과 같은 claim이 들어갈 수 있다.

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

JWT는 `ClaimsPrincipal`과 같은 객체가 아니며, 모든 `ClaimsPrincipal`이 JWT에서 만들어지는 것도 아니다. Cookie, client certificate, 외부 로그인 또는 custom authentication handler도 `ClaimsPrincipal`을 만들 수 있다.

### Claim

`Claim`은 인증된 주체에 관한 하나의 type/value 정보다.

```cs
new Claim("sub", "user-123");
new Claim("department", "Engineering");
new Claim("role", "Admin");
```

Claim은 데이터일 뿐이다. Claim이 있다고 해서 자동으로 접근 권한이 부여되는 것은 아니다. 인가 규칙이 해당 Claim을 어떻게 해석할지 결정한다.

### ClaimsIdentity

`ClaimsIdentity`는 Claim 묶음과 인증 방식에 관한 정보를 가진다.

```cs
var identity = new ClaimsIdentity(
    claims,
    authenticationType: "Bearer",
    nameType: "name",
    roleType: "role");
```

여기서 `nameType`과 `roleType`은 어떤 Claim을 이름과 역할로 해석할지 지정한다.

### ClaimsPrincipal

`ClaimsPrincipal`은 하나 이상의 `ClaimsIdentity`를 포함한다. 한 주체가 여러 인증 수단에서 얻은 identity를 함께 가질 수 있기 때문이다.

```cs
var principal = new ClaimsPrincipal(identity);
```

### HttpContext.User

ASP.NET Core에서 `HttpContext.User` 또는 Controller의 `User`는 별도의 사용자 entity가 아니다. 현재 요청의 `ClaimsPrincipal`을 가리키는 property다.

따라서 다음 코드는 token이나 Cookie를 직접 읽는 것이 아니라 인증 handler가 만든 `ClaimsPrincipal`을 조회한다.

```cs
var subject = User.FindFirst("sub")?.Value;
var authenticated = User.Identity?.IsAuthenticated == true;
var isAdmin = User.IsInRole("Admin");
```

Identity의 `ApplicationUser`와 `HttpContext.User`도 구분해야 한다. `ApplicationUser`는 데이터베이스에 저장된 사용자 entity이고, `HttpContext.User`는 현재 요청에 포함된 보안 context다.

## Role 기반 인가는 왜 Claim 기반인가?

ASP.NET Core runtime에서 역할은 지정된 role claim type을 가진 Claim으로 표현된다.

```cs
[Authorize(Roles = "Admin")]
public IActionResult AdminOnly() => Ok();
```

위 검사는 개념적으로 다음과 같은 질문을 한다.

```text
현재 ClaimsPrincipal의 인증된 ClaimsIdentity 중
RoleClaimType에 해당하며 값이 "Admin"인 Claim이 있는가?
```

기본 `RoleClaimType`은 `ClaimTypes.Role`이지만 항상 그 값만 사용하는 것은 아니다. JWT의 원래 이름인 `role`을 사용하도록 설정할 수도 있다.

```cs
options.TokenValidationParameters = new TokenValidationParameters
{
    RoleClaimType = "role",
    NameClaimType = "name"
};
```

따라서 “role은 무조건 `ClaimTypes.Role` Claim이다”보다는 다음 표현이 더 정확하다.

> ASP.NET Core는 `ClaimsIdentity.RoleClaimType`으로 지정된 Claim들을 role로 해석한다.

## Claim 기반 인가와 Policy 기반 인가

Policy는 ASP.NET Core 인가 기능의 일반적인 실행 모델이다. Policy는 하나 이상의 requirement로 구성되고, authorization handler가 각 requirement를 평가한다.

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

Role 검사와 Claim 검사 모두 policy engine에서 requirement로 실행된다.

```text
Policy 기반 인가
  ├── Role requirement
  ├── Claim requirement
  ├── 인증 여부 requirement
  └── custom requirement + handler
```

그래서 “Claim 기반 인가는 Policy 기반 인가다”라고 동일시하기보다는 다음과 같이 이해하는 것이 정확하다.

> Policy는 인가 규칙을 실행하는 일반적인 framework이고, role과 claim 검사는 그 framework가 제공하는 대표적인 requirement다.

복잡한 규칙은 단일 Claim 확인만으로 표현하기 어렵다. 예를 들어 “현재 사용자가 이 tenant에 속하며, 이 주문의 소유자이거나 지원 담당자인가?” 같은 규칙은 resource-based authorization이나 custom requirement가 적합하다.

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

## ClaimTypes와 JWT/OIDC claim 이름이 다른 이유

.NET의 `System.Security.Claims.ClaimTypes`는 well-known claim type을 문자열 상수로 제공한다.

```cs
ClaimTypes.Name
// http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name

ClaimTypes.NameIdentifier
// http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier

ClaimTypes.Role
// http://schemas.microsoft.com/ws/2008/06/identity/claims/role
```

### ClaimTypes의 긴 문자열은 XML namespace URI인가?

그렇다. `ClaimTypes`의 긴 문자열은 XML/SOAP 시대의 **XML namespace-style schema URI**를 claim type identifier로 사용한 것이다. WIF(Windows Identity Foundation)와 WS-Federation 같은 WS-* 기술은 서로 다른 표준과 공급자가 정의한 이름이 충돌하지 않도록 URI namespace를 사용했다. `ClaimTypes`는 이 naming convention을 물려받았다.

다만 이것이 XML 문서의 `xmlns` 선언 그 자체라는 뜻은 아니다. `ClaimTypes.Name` 같은 값은 .NET `Claim`의 `Type`을 식별하는 문자열이며, XML namespace와 동일한 URI namespacing 방식을 사용하는 **schema URI identifier**라고 이해하는 것이 정확하다.

```xml
<!-- XML namespace도 URI를 namespace identifier로 사용한다. -->
<claims xmlns="http://schemas.xmlsoap.org/ws/2005/05/identity/claims">
    <name>Alice</name>
</claims>
```

```cs
// .NET에서는 전체 URI가 Claim.Type 문자열이다.
var claim = new Claim(
    "http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name",
    "Alice");

Debug.Assert(claim.Type == ClaimTypes.Name);
```

`http://`로 시작해 URL처럼 보이지만 실제 network resource를 가리키거나 browser로 접속할 수 있어야 하는 것은 아니다. 여기서 URI의 목적은 위치를 알려주는 것이 아니라 전역적으로 구분 가능한 이름을 제공하는 것이다. Microsoft 문서도 이 값들을 well-known claim type의 URI라고 설명하며 schema URI가 hyperlink로 동작할 것으로 기대하면 안 된다고 명시한다.

이 값들은 JWT보다 먼저 만들어졌기 때문에 OIDC/JWT의 짧은 JSON claim name과 직접 일치하지 않는다.

| 의미 | .NET `ClaimTypes` URI | JWT/OIDC 이름 |
|---|---|---|
| 사용자 식별자 | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier` | `sub` |
| 표시 이름 | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/name` | `name` |
| 이메일 | `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/emailaddress` | `email` |
| 역할 | `http://schemas.microsoft.com/ws/2008/06/identity/claims/role` | 구현에 따라 `role` 또는 `roles` |

반면 JWT와 OpenID Connect는 JSON에 적합한 짧은 이름을 사용한다.

### “Reserved claim”보다 “Registered Claim Name”이 정확하다

JWT claim을 설명할 때 흔히 “reserved claim”이라고 부르지만 RFC 7519의 공식 용어는 **Registered Claim Name**이다. “reserved”라고 하면 해당 이름을 반드시 사용해야 하거나 JWT마다 반드시 포함해야 한다고 오해하기 쉽다.

RFC 7519는 claim name을 세 종류로 구분한다.

| 분류 | 의미 | 예시 |
|---|---|---|
| Registered | 여러 시스템이 같은 의미로 사용할 수 있도록 IANA registry에 등록된 이름 | `iss`, `sub`, `aud`, `exp` |
| Public | 충돌 방지를 위해 공개적으로 등록하거나 collision-resistant name을 사용한 이름 | `https://example.com/claims/tenant_id` |
| Private | 발급자와 소비자가 합의해서 사용하는 이름. 전역적인 의미나 충돌 방지 보장이 없음 | `tenant_id`, `permission` |

중요한 점은 **RFC 7519 자체는 registered claim을 모든 JWT에 의무화하지 않는다**는 것이다. 어떤 claim이 필수인지는 JWT를 사용하는 상위 protocol이나 애플리케이션 profile이 결정한다.

예를 들어 일반 JWT에서는 `exp`가 선택 사항이지만, OpenID Connect ID token에서는 `exp`가 필수다. “JWT 표준 claim이므로 필수”가 아니라 “OIDC ID token 규칙이 그 JWT에 `exp`를 요구한다”가 정확한 설명이다.

### RFC 7519의 기본 Registered Claim Names

RFC 7519가 처음 등록한 일곱 claim은 다음과 같다.

| Claim | 이름 | 값의 형태 | 의미와 검증 포인트 |
|---|---|---|---|
| `iss` | Issuer | case-sensitive string 또는 URI | token 발급자를 식별한다. 신뢰하는 issuer와 정확히 일치하는지 확인해야 한다. |
| `sub` | Subject | case-sensitive string 또는 URI | token이 설명하는 주체다. issuer 범위 안에서 고유해야 한다. 사용자 ID일 수도 있고 client credentials flow에서는 client를 나타낼 수도 있다. |
| `aud` | Audience | string 또는 string 배열 | token을 받을 대상을 식별한다. 현재 API가 audience에 없다면 token을 거부해야 한다. |
| `exp` | Expiration Time | NumericDate | 이 시각 이후에는 token을 받아들이면 안 된다. 검증 시 제한된 clock skew를 둘 수 있다. |
| `nbf` | Not Before | NumericDate | 이 시각 전에는 token을 받아들이면 안 된다. |
| `iat` | Issued At | NumericDate | token이 발급된 시각이다. token 나이 계산 등에 사용할 수 있지만 이것만으로 만료를 판단하지 않는다. |
| `jti` | JWT ID | case-sensitive string | token의 고유 식별자다. replay 방지나 revocation record에 사용할 수 있지만 저장하고 검사하는 구현이 없으면 자동으로 replay를 막아주지는 않는다. |

`NumericDate`는 UTC 기준 1970-01-01부터 흐른 초를 나타내는 JSON number다. ISO 8601 문자열이 아니다.

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

이 값들은 단순한 사용자 profile 정보가 아니라 token의 보안 context를 구성한다. 특히 signature만 검증하고 `iss`, `aud`, `exp`를 검증하지 않으면 “정상적으로 서명되었지만 다른 API를 위해 발급된 token”을 잘못 받을 수 있다.

### IANA registry는 RFC 7519의 일곱 개로 끝나지 않는다

IANA JSON Web Token Claims registry에는 다른 표준이 이후 등록한 이름도 포함된다. 예를 들면 다음과 같다.

- OIDC 사용자 정보: `name`, `email`, `email_verified`, `picture`
- 인증 정보: `auth_time`, `nonce`, `acr`, `amr`, `azp`
- OAuth 정보: `client_id`, `scope`
- 인가 정보: `groups`, `roles`, `entitlements`

따라서 “registered claim”은 RFC 7519의 일곱 claim만을 의미하지 않는다. 현재 registry에 등록된 전체 이름 집합을 가리키며, 각 claim의 의미와 요구 조건은 해당 claim을 등록한 specification에서 확인해야 한다.

여기서 단수 `role`과 복수 `roles`도 구분해야 한다. RFC 9068의 JWT access token profile은 권한 정보를 표현할 때 `groups`, `roles`, `entitlements`를 권장한다. 반면 여러 .NET 또는 identity provider 구현은 관례적으로 `role`이나 Microsoft URI 형식의 `ClaimTypes.Role`을 사용한다. 실제 token의 claim name과 ASP.NET Core의 `RoleClaimType`을 일치시켜야 한다.

### OpenID Connect ID token의 claim

ID token은 “OAuth access token에 사용자 정보를 조금 더 넣은 것”이 아니다. 클라이언트가 authorization server에서 발생한 사용자 인증 결과를 확인하기 위한 별도의 token이며 항상 JWT 형식이다.

OIDC Core가 ID token에 요구하는 핵심 claim은 다음과 같다.

| Claim | 요구 조건 | 의미 |
|---|---|---|
| `iss` | 필수 | ID token을 발급한 OpenID Provider |
| `sub` | 필수 | issuer 안에서 고유하고 재할당되지 않는 사용자 식별자 |
| `aud` | 필수 | 이 ID token을 받을 client. 보통 `client_id`를 포함한다. |
| `exp` | 필수 | ID token 만료 시각 |
| `iat` | 필수 | ID token 발급 시각 |
| `auth_time` | 요청 조건에 따라 필수 | 사용자가 실제로 인증된 시각 |
| `nonce` | authorization request에 보냈다면 필수 | 요청과 응답을 연결하고 replay 공격을 방지한다. client가 원래 값과 비교해야 한다. |
| `acr` | 선택 | 인증 context class. 인증 강도나 정책을 나타낼 수 있다. |
| `amr` | 선택 | 실제 사용한 인증 방법 목록. 예: password, OTP |
| `azp` | 특정 조건에서 사용 | ID token이 발급된 authorized party. audience가 여러 개일 때 특히 중요하다. |

`sub`는 로그인 이름이나 이메일 주소로 사용하기보다 안정적인 내부 식별자로 취급해야 한다. 이메일은 변경되거나 재사용될 수 있지만 OIDC `sub`는 issuer 범위에서 사용자를 안정적으로 식별하도록 설계되었다. 데이터베이스에서 외부 사용자를 연결할 때는 보통 `(iss, sub)` 조합을 key로 사용한다. 서로 다른 issuer가 같은 `sub` 값을 발급할 수 있기 때문이다.

OIDC의 Standard Claims는 사용자 profile을 표현한다.

| 영역 | Claim |
|---|---|
| 이름 | `name`, `given_name`, `family_name`, `middle_name`, `nickname`, `preferred_username` |
| profile | `profile`, `picture`, `website` |
| 연락처 | `email`, `email_verified`, `phone_number`, `phone_number_verified` |
| 기타 | `gender`, `birthdate`, `zoneinfo`, `locale`, `address`, `updated_at` |

이 claim들은 정의되어 있다고 해서 항상 ID token에 들어가는 것이 아니다. 요청한 scope, `claims` parameter, 사용자 동의, provider 정책에 따라 ID token이나 UserInfo response에서 제공될 수 있다.

대표적인 scope와 claim의 관계는 다음과 같다.

- `openid`: 요청을 OIDC 요청으로 만든다. 사용자 profile 전체를 의미하지 않는다.
- `profile`: 이름, 사진, locale, `updated_at` 등의 profile claim을 요청한다.
- `email`: `email`, `email_verified`를 요청한다.
- `phone`: `phone_number`, `phone_number_verified`를 요청한다.
- `address`: `address`를 요청한다.

scope는 “반드시 이 claim을 token에 넣으라”는 단순 명령이 아니다. client가 요청하는 접근 범위를 나타내며, 실제 결과는 consent와 provider 정책의 영향을 받는다.

### ID token과 access token의 claim을 혼동하지 않기

ID token의 audience는 **client application**이고, access token의 audience는 일반적으로 **resource server/API**다.

```text
ID token
  issuer ──인증 결과──> client application

Access token
  issuer ──API 접근 권한──> resource server
```

따라서 API는 ID token을 access token 대신 받아서는 안 된다. 두 token이 모두 JWT이고 비슷한 claim을 포함해도 목적과 audience가 다르다.

또한 OAuth access token은 반드시 JWT일 필요가 없다. OpenIddict도 opaque access token을 사용할 수 있다. JWT 형식의 access token이 RFC 9068 profile을 따른다면 다음 claim들이 필수다.

- `iss`
- `sub`
- `aud`
- `exp`
- `iat`
- `jti`
- `client_id`

authorization request에 scope가 있었다면 RFC 9068은 access token에 `scope` claim을 포함할 것을 권고한다. 그러나 client는 access token 내부 구조에 의존하지 말고 opaque 값으로 취급해야 한다. access token의 소비자는 API이며, client가 token payload를 읽어서 UI 권한이나 로그인 상태를 결정하면 token format 변경에 취약해진다.

### Custom claim 이름을 정하는 방법

내부 시스템에서만 사용하는 claim은 짧은 private name을 쓸 수 있다.

```json
{
  "tenant_id": "tenant-7",
  "permission": ["orders.read", "orders.write"]
}
```

하지만 여러 조직이나 제품 사이에서 token을 교환한다면 짧은 이름은 미래의 표준 또는 다른 공급자의 claim과 충돌할 수 있다. 이런 경우 자신이 통제하는 namespace를 사용한 collision-resistant name이 안전하다.

```json
{
  "https://auth.example.com/claims/tenant_id": "tenant-7"
}
```

URI 형식 claim name도 실제로 접속 가능한 URL일 필요는 없다. 자신이 통제하는 namespace에 속한 고유한 식별자라는 점이 핵심이다. 참고로 RFC 7519의 `StringOrURI` 규칙은 `iss`나 `sub` 같은 일부 **claim value**의 자료형에 관한 규칙이며 모든 claim name에 적용되는 규칙은 아니다.

Custom claim을 설계할 때는 다음을 확인하는 것이 좋다.

1. 같은 의미의 registered claim이 IANA registry에 이미 있는가?
2. 이 claim의 소비자는 client인가, API인가?
3. 값의 형식이 string, array, boolean 중 무엇인지 계약으로 정했는가?
4. 이름 충돌 가능성이 있다면 collision-resistant name을 사용했는가?
5. token에 넣지 않아도 API가 서버에서 조회할 수 있는 민감 정보인가?
6. 값이 변경되었을 때 이미 발급된 token이 오래된 권한을 유지해도 되는가?

JWT payload는 암호화되지 않은 JWS인 경우 누구나 base64url decode해서 읽을 수 있다. 서명은 위변조를 막지만 기밀성을 제공하지 않는다. 비밀번호, secret, 불필요한 개인정보를 claim에 넣으면 안 된다. 권한 정보를 넣을 때도 token 만료 전까지 값이 stale할 수 있다는 점을 고려해야 한다.

## Inbound claim mapping

JWT bearer handler는 token의 짧은 claim 이름을 .NET의 URI 형식 claim type으로 mapping할 수 있다. 예를 들어 `sub`가 `ClaimTypes.NameIdentifier`로 변환될 수 있다.

이 동작은 scheme별로 끌 수 있다.

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

mapping을 끄면 `User.FindFirst("sub")`처럼 token에 들어 있는 이름을 그대로 사용할 수 있다. 중요한 것은 mapping을 켜거나 끄는 선택 자체보다 애플리케이션 전체에서 claim 이름과 `NameClaimType`, `RoleClaimType`을 일관되게 사용하는 것이다.

## Identity의 bearer token과 OAuth token은 다르다

최근 ASP.NET Core Identity API endpoint는 first-party client를 위해 bearer token을 발급할 수 있다. 그러나 이 token은 Identity 전용 proprietary token이며 OAuth 2.0/OIDC token server를 구현한 것이 아니다.

다음 기능이 필요하다면 OpenIddict 같은 OAuth/OIDC server framework가 필요하다.

- 여러 client application 등록
- authorization code + PKCE
- 표준 scope와 consent
- ID token과 OIDC discovery
- refresh token 정책
- token introspection과 revocation
- 외부 애플리케이션과의 표준 기반 상호 운용

사용자가 있는 신규 애플리케이션에서는 password grant보다 authorization code flow with PKCE를 우선해야 한다. password grant는 client가 사용자의 비밀번호를 직접 받기 때문에 권장되지 않는다. 서버 간 통신에는 client credentials flow가 적합하다.

## 전체 구조 정리

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

마지막으로 각 용어를 한 문장으로 정리하면 다음과 같다.

- Identity: 사용자 계정과 로그인 lifecycle을 관리한다.
- OpenIddict: OAuth 2.0/OIDC protocol을 처리하고 표준 token을 발급하거나 검증한다.
- JWT: claim을 전달할 수 있는 signed token format이다.
- Claim: 주체에 관한 type/value 정보다.
- ClaimsIdentity: 하나의 인증 출처에서 얻은 Claim 묶음이다.
- ClaimsPrincipal: 하나 이상의 ClaimsIdentity를 포함하는 현재 보안 주체다.
- `HttpContext.User`: 현재 요청의 ClaimsPrincipal이다.
- Role: RoleClaimType으로 지정된 Claim을 ASP.NET Core가 role로 해석한 것이다.
- Policy: role, claim, custom requirement를 평가하는 인가 규칙이다.

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
