+++
title = "IdentityServer UserClaims 이해하기: User, Identity Resource, API Resource의 차이"
date = 2026-07-06

[taxonomies]
categories = ["post"]
tags = ["C#", "ASP.NET Core", "IdentityServer", "OAuth 2.0", "OpenID Connect", "Claims"]
+++

IdentityServer 관리 화면을 처음 보면 `UserClaims`라는 이름이 여러 곳에 반복해서 등장한다.

- Users/Roles의 User에 있는 UserClaims
- Identity Resource의 UserClaims
- API Resource의 UserClaims
- 버전에 따라 API Scope에도 있는 UserClaims

이름은 같지만 같은 데이터를 저장하는 필드는 아니다. 이 차이를 놓치면 Identity Resource가 사용자의 Claim을 소유한다고 생각하거나, User 자체도 Resource라고 생각하기 쉽다.

핵심은 한 문장으로 정리할 수 있다.

> User의 UserClaims는 실제 type/value 데이터이고, Resource의 UserClaims는 token에 필요한 claim type을 고르는 선택 목록이다.

## Claim은 type과 value로 이루어진다

Claim은 어떤 주체에 관한 한 가지 사실이다.

```text
type = department
value = Engineering
```

```text
type = omni_role
value = platform_admin
```

`department`와 `omni_role`은 claim type이고, `Engineering`과 `platform_admin`은 해당 사용자의 claim value다.

.NET에서는 다음처럼 표현한다.

```cs
new Claim("department", "Engineering");
new Claim("omni_role", "platform_admin");
```

Claim type은 문자열이다. `role`도 token 안에서는 다른 Claim과 같은 문자열 type이다. 어떤 Claim을 이름이나 role로 해석할지는 Claim 자체가 아니라 IdentityServer의 발급 과정과 ASP.NET Core의 소비 설정이 결정한다.

## 세 곳의 UserClaims는 역할이 다르다

같은 `UserClaims`라는 이름을 다음처럼 구분해야 한다.

| 위치 | 저장하는 것 | 역할 |
|---|---|---|
| User → UserClaims | type과 value | 특정 사용자에게 속한 실제 데이터 |
| Identity Resource → UserClaims | type 이름 | identity scope가 요청할 사용자 정보 목록 |
| API Resource / API Scope → UserClaims | type 이름 | access token에 필요하다고 요청할 사용자 정보 목록 |

첫 번째는 **데이터**다. 나머지 두 개는 **선택자(selector)** 또는 **포장 목록(packing list)** 이다.

## User의 UserClaims는 실제 사용자별 데이터다

Alice에게 다음 Claim이 저장되어 있다고 하자.

```text
User: Alice

department = Engineering
omni_role  = platform_admin
locale     = ko-KR
```

Bob은 다른 값을 가질 수 있다.

```text
User: Bob

department = Support
omni_role  = platform_member
locale     = en-US
```

이 값들은 사용자별 데이터다. ASP.NET Core Identity를 저장소로 사용한다면 raw user claim은 일반적으로 `AspNetUserClaims` 같은 별도 table에 저장된다. 따라서 "User entity에 붙은 Claim"이라고 이해할 수 있지만, 반드시 User table의 column이라는 뜻은 아니다.

같은 type에 여러 value를 갖는 것도 가능하다.

```text
role = Admin
role = Auditor
```

Resource 설정에는 `role`을 한 번만 적는다. 그 설정은 특정 value 하나가 아니라 `role`이라는 type을 선택하기 때문이다. Profile Service가 두 값을 모두 발급하면 token에서는 배열 또는 여러 Claim으로 표현될 수 있다.

```json
{
  "role": ["Admin", "Auditor"]
}
```

완전히 같은 type/value 쌍을 중복 저장하는 것은 의미가 없으므로 피하는 것이 좋다.

## Identity Resource의 UserClaims는 identity claim type 목록이다

Identity Resource는 사용자 한 명을 나타내지 않는다. 클라이언트가 OIDC `scope` parameter로 요청할 수 있는 **이름이 붙은 identity 정보 묶음**이다.

예를 들어 다음 Identity Resource는 `name`, `email`, `locale` type을 묶는다.

```cs
new IdentityResource(
    name: "profile",
    userClaims: new[] { "name", "email", "locale" },
    displayName: "User profile");
```

클라이언트는 다음처럼 `profile` scope를 요청한다.

```text
scope=openid profile
```

여기서 Identity Resource의 UserClaims에는 value가 없다.

```text
name
email
locale
```

이 목록의 뜻은 다음과 같다.

> 현재 로그인한 사용자의 `name`, `email`, `locale` 값을 Profile Service에 요청하라.

Alice가 `locale = ko-KR`을 가지고 있으면 Alice의 값이 발급된다. Bob이 `locale = en-US`를 가지고 있으면 같은 Identity Resource를 요청해도 Bob의 값이 발급된다. Resource에 `ko-KR`이나 `en-US`가 저장되는 것은 아니다.

Identity Resource에서 선택된 Claim은 identity token 또는 userinfo 처리에 사용된다. 정확히 어느 응답에 포함되는지는 flow와 IdentityServer 설정에 따라 달라질 수 있지만, API 호출 권한을 나타내는 API Scope와는 목적이 다르다.

## API Resource의 UserClaims는 access token용 claim type 목록이다

API Resource는 보호할 논리적 API를 나타낸다. 일반적으로 access token의 audience와 관련되며 여러 API Scope를 묶을 수 있다.

```cs
var omniApi = new ApiResource("omniapi", "Omni API")
{
    Scopes = { "omniapi.read", "omniapi.write" },
    UserClaims = { "omni_role" }
};
```

이 설정은 `omni_role` 값을 API Resource 안에 저장한다는 의미가 아니다.

> `omniapi`용 access token을 만들 때 현재 사용자의 `omni_role` Claim을 요청하라.

Alice의 사용자 데이터에 다음 값이 있다면,

```text
omni_role = platform_admin
```

Profile Service가 해당 값을 읽어 access token에 넣을 수 있다.

```json
{
  "sub": "alice-id",
  "aud": "omniapi",
  "scope": "omniapi.read",
  "omni_role": "platform_admin"
}
```

API Scope에도 UserClaims를 설정할 수 있다. 차이는 적용 단위다.

- API Resource UserClaims: Resource에 속한 scope 전반에서 공통으로 필요한 Claim
- API Scope UserClaims: 특정 scope를 요청했을 때만 필요한 Claim

예를 들어 `omniapi.write`에서만 `approval_level`이 필요하다면 API Scope에 둘 수 있다.

```cs
new ApiScope(
    name: "omniapi.write",
    displayName: "Write access",
    userClaims: new[] { "approval_level" });
```

## Profile Service가 데이터와 선택 목록을 연결한다

Resource 설정이 Claim value를 직접 읽어 token에 복사하는 것은 아니다. IdentityServer는 Claim이 필요할 때 Profile Service를 호출한다.

Profile Service가 받는 중요한 정보는 다음과 같다.

- 어떤 client가 요청했는가
- 현재 user는 누구인가
- identity token, access token, userinfo 중 어떤 요청인가
- 어떤 claim type들이 요청되었는가

단순화하면 다음과 같은 흐름이다.

```text
Client가 scope/resource 요청
        ↓
IdentityServer가 Resource 설정에서 claim type 수집
        ↓
Profile Service에 RequestedClaimTypes 전달
        ↓
Profile Service가 현재 User의 실제 값 조회 또는 계산
        ↓
허용된 Claim을 token/userinfo에 발급
```

개념적인 코드는 다음과 같다.

```cs
public async Task GetProfileDataAsync(ProfileDataRequestContext context)
{
    var userClaims = await LoadClaimsForCurrentUser(context.Subject);

    context.IssuedClaims.AddRange(
        userClaims.Where(claim =>
            context.RequestedClaimTypes.Contains(claim.Type)));
}
```

실제 구현에서는 user 활성 상태, client별 공개 범위, 동의, 민감 정보, 중복 Claim 등을 함께 처리해야 한다. 중요한 점은 Profile Service가 연결 지점이라는 것이다.

또한 Claim value가 반드시 Users/Roles 화면의 UserClaims에서 올 필요는 없다. Profile Service는 여러 source를 사용할 수 있다.

- raw user claims
- managed role assignment
- 별도 application database
- LDAP 또는 외부 identity provider
- 요청 시 계산한 값

Resource의 UserClaims는 source를 지정하지 않는다. 필요한 **type**만 지정한다.

## User가 Resource는 아니다

IdentityServer의 용어에서 User와 Resource는 역할이 다르다.

- User: client를 사용해 로그인하고 Resource에 접근하는 사람, 즉 token의 subject
- Client: token을 요청하는 application
- Identity Resource: 요청 가능한 사용자 identity 정보 묶음
- API Resource: 보호할 API 또는 기능

OAuth 용어로 User를 resource owner라고 부르기도 한다. 하지만 resource owner와 IdentityServer 설정 객체인 Resource는 같은 뜻이 아니다.

```text
User ──사용──▶ Client ──token 요청──▶ IdentityServer
  │                                      │
  └──────── Resource에 접근 ◀────────────┘
```

Identity Resource가 사용자 Claim을 묶기 때문에 User 자체도 Resource처럼 보일 수 있다. 그러나 Identity Resource가 나타내는 것은 User 객체가 아니라, client가 요청할 수 있도록 정의한 **사용자 정보의 일부**다.

예를 들어 `profile`은 Alice라는 사용자가 아니다.

```text
profile = [name, email, locale]이라는 요청 가능한 schema
Alice   = 그 schema에 실제 값을 제공하는 subject
```

## 어디까지 표준이고 어디부터 IdentityServer 구현인가?

IdentityServer가 Claim 개념 전체를 발명한 것은 아니다. OAuth 2.0, OpenID Connect, JWT가 정의한 protocol 위에 IdentityServer가 자신의 configuration과 확장 model을 제공한다.

| 구조 | 표준 여부 | 정의한 곳 |
|---|---|---|
| JWT claim set과 `iss`, `sub`, `aud`, `exp` | 표준 | RFC 7519 |
| ID token, UserInfo, `openid`, `profile`, `email` scope | 표준 | OpenID Connect |
| `IdentityResource`, `ApiResource`, `UserClaims` collection | 제품 구현 | Duende IdentityServer |
| `ProfileService`와 store interface | 제품 구현 | Duende IdentityServer |
| `Claim`, `ClaimsIdentity`, `ClaimsPrincipal` | framework 구현 | .NET |
| `AspNetUserClaims`, `AspNetUserRoles` table | framework 구현 | ASP.NET Core Identity |

표준은 주로 system 사이에서 교환할 message와 의미를 정의한다.

```text
Identity Provider
    │
    │ 표준화된 protocol/message
    │ ID token, UserInfo, JWT claim set
    ▼
Client / API
```

예를 들어 JWT는 JSON claim set의 표현 방법과 registered claim name을 정의한다.

```json
{
  "iss": "https://identity.example.com",
  "sub": "alice-id",
  "aud": "omniapi",
  "exp": 1783378800,
  "omni_role": "platform_admin"
}
```

`iss`, `sub`, `aud`, `exp`는 JWT registered claim이다. `omni_role`은 application이 정의한 private claim이다. JWT는 registered, public, private claim name을 모두 허용한다.

OpenID Connect는 `sub`, `name`, `email`, `picture` 같은 standard claim과 이를 요청하는 `openid`, `profile`, `email` 등의 scope를 정의한다. 그러나 `IdentityResource`라는 C# class나 관리 화면의 UserClaims field는 정의하지 않는다.

IdentityServer의 `IdentityResource`는 다음 표준 동작을 구현하기 위한 configuration abstraction이다.

```text
OIDC 표준 개념
  profile scope → name, family_name, picture 등의 Claim 요청

IdentityServer 구현
  IdentityResource("profile")
      UserClaims = [name, family_name, picture]
```

API Resource와 Profile Service도 마찬가지다. 보호할 API, audience, scope, 사용자 Claim 발급이라는 protocol 요구사항은 표준에서 오지만, 이를 `ApiResource` class와 `IProfileService` interface로 구성하는 방식은 IdentityServer의 설계다.

또한 OAuth access token이 항상 JWT인 것은 아니다. Authorization server는 opaque access token을 발급할 수도 있다. JWT를 사용하는 경우에만 API가 JSON claim set을 직접 볼 수 있다.

## Claim 저장 구조는 표준화되어 있는가?

OAuth, OpenID Connect, JWT는 IdP 내부 database schema를 규정하지 않는다. 다음 중 무엇을 사용해도 token이 protocol 규칙을 지키면 상호 운용할 수 있다.

```text
Identity Provider 내부
  ├── relational database
  ├── ASP.NET Core Identity table
  ├── LDAP / Active Directory
  ├── 외부 identity provider
  ├── 별도 user-management API
  └── 요청 시 계산한 Claim
```

따라서 다음 table 구조는 표준이 아니다.

```text
AspNetUsers
AspNetUserClaims
AspNetRoles
AspNetUserRoles
```

다른 IdP는 완전히 다른 schema를 사용할 수 있다. IdentityServer도 configuration, resource, persisted grant 등을 위한 store interface를 제공하며, in-memory store나 custom database implementation을 선택할 수 있다. User 저장소 역시 ASP.NET Core Identity로 고정되지 않는다.

표준이 내부 저장소를 강제하지 않는 이유는 Claim source가 다양하기 때문이다. `department`는 HR database에서, `role`은 directory group에서, `email`은 external IdP에서, 요청 시 계산하는 `risk_level`은 별도 service에서 올 수 있다. Profile Service는 이런 source를 token claim으로 projection하는 IdentityServer의 연결 지점이다.

## SCIM은 무엇을 표준화하는가?

사용자와 group을 system 사이에서 provisioning하기 위한 별도 표준으로 SCIM(System for Cross-domain Identity Management)이 있다.

SCIM은 다음을 정의한다.

- User와 Group의 JSON resource schema
- `userName`, `name`, `emails`, `groups`, `roles`, `entitlements` 같은 attribute
- schema extension과 namespace
- HTTP를 이용한 생성, 조회, 수정, 삭제 protocol

SCIM User는 다음과 같은 형태다.

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

SCIM은 여러 system이 User와 Group 데이터를 같은 형식으로 provisioning하게 해준다. 하지만 다음은 여전히 결정하지 않는다.

- 실제 database table을 어떻게 설계할지
- SCIM attribute를 어떤 token claim으로 변환할지
- `Admin` role이 실제로 어떤 권한을 부여할지
- IdentityServer의 어느 Resource가 해당 Claim을 요청할지

즉 SCIM은 **user-management API와 교환 schema**를 표준화하지만, IdP의 물리적 저장 구조나 token 발급 정책까지 표준화하지 않는다.

전체 경계를 정리하면 다음과 같다.

```text
SCIM
  User/Group provisioning과 교환 schema를 표준화
        ↓
IdP 내부 저장소
  제품 또는 조직이 자유롭게 설계
        ↓
IdentityServer configuration + Profile Service
  저장된 정보를 token Claim으로 projection
        ↓
OAuth / OpenID Connect / JWT
  외부로 전달되는 protocol과 message를 표준화
        ↓
.NET Claim / ClaimsIdentity / ClaimsPrincipal
  API process 내부의 runtime object로 변환
```

## `role`은 예약된 Claim type인가?

`role`은 token 형식 수준에서 예약된 특별한 자료형이 아니다. 문자열 claim type이다.

```cs
new Claim("role", "Admin");
```

하지만 실제 system에서는 관례적으로 특별하게 취급될 수 있다.

1. Identity provider의 Role 관리 기능이 role assignment를 `role` Claim으로 변환할 수 있다.
2. Resource가 `role` type을 요청하면 그 값들이 token에 들어갈 수 있다.
3. ASP.NET Core가 `role`을 `RoleClaimType`으로 지정하면 `IsInRole()`이 그 Claim을 role로 해석한다.

```cs
options.TokenValidationParameters.RoleClaimType = "role";
```

```cs
User.IsInRole("Admin");
```

위 코드는 개념적으로 현재 `ClaimsIdentity`에 `type = role`, `value = Admin`인 Claim이 있는지 검사한다.

따라서 `role`은 기술적으로 임의의 문자열이지만, system이 role namespace로 채택하면 운영상 소유권이 있는 이름처럼 다뤄야 한다.

## raw user claim과 managed role assignment

사용자에게 role 정보를 부여하는 방법은 두 가지가 있을 수 있다.

### raw user claim

```text
UserClaims
  type  = role
  value = Admin
```

### managed role assignment

```text
Roles
  Admin

UserRoles
  Alice → Admin
```

저장 구조는 다르지만 Profile Service가 둘을 같은 방식으로 projection하면 token 결과는 같을 수 있다.

```json
{
  "role": "Admin"
}
```

managed role은 role 정의, 사용자 assignment, group inheritance, 관리 UI 같은 구조를 제공한다. raw user claim은 단순하고 유연하지만 typo와 잘못된 value를 통제하기 어렵다.

두 방식을 동시에 사용한다면 중복 또는 충돌 규칙을 명확히 해야 한다.

```text
Role assignment → role = Admin
Raw user claim  → role = Support
```

둘 다 발급되면 API 입장에서는 사용자가 `Admin`과 `Support` role을 모두 가진다. API는 어느 값이 어느 저장 source에서 왔는지 token만 보고 구분할 수 없다.

## 공유 IdP에서는 claim type을 namespace로 분리한다

여러 제품이 하나의 IdP를 공유한다면 generic한 `role`은 충돌하기 쉽다.

```text
role = Admin
```

이 값만 보면 어느 제품의 Admin인지 알 수 없다. 각 제품이 같은 type과 value를 사용하면 다른 제품의 role이 잘못된 권한으로 해석될 수 있다.

제품별 type을 사용하면 경계가 명확해진다.

```text
omni_role    = platform_admin
billing_role = auditor
support_role = agent
```

Omni API는 다음과 같이 자신의 type만 role로 지정할 수 있다.

```cs
options.TokenValidationParameters.RoleClaimType = "omni_role";
```

그러면 다음 검사는 `omni_role`만 확인한다.

```cs
User.IsInRole("platform_admin");
```

value도 `Admin`이나 `Manager`처럼 문맥이 부족한 이름보다 `platform_admin`, `platform_member`처럼 의미가 드러나는 이름이 안전하다. Claim value 비교는 일반적으로 정확한 문자열 일치를 사용하므로 대소문자와 spelling도 계약의 일부로 관리해야 한다.

## 전체 예제로 다시 보기

### 1. User에 실제 값을 저장한다

```text
Alice
  sub       = alice-id
  name      = Alice
  email     = alice@example.com
  omni_role = platform_admin
```

### 2. Identity Resource는 identity claim type을 고른다

```text
Identity Resource: profile
UserClaims: name, email
```

### 3. API Resource는 access token claim type을 고른다

```text
API Resource: omniapi
Scopes: omniapi.read, omniapi.write
UserClaims: omni_role
```

### 4. Client가 요청한다

```text
scope=openid profile omniapi.read
```

### 5. Profile Service가 Alice의 값을 찾는다

```text
요청된 identity type: name, email
요청된 API type: omni_role

Alice의 값:
  name = Alice
  email = alice@example.com
  omni_role = platform_admin
```

### 6. API는 access token만 본다

```json
{
  "sub": "alice-id",
  "aud": "omniapi",
  "scope": "omniapi.read",
  "omni_role": "platform_admin"
}
```

API에 도착하면 Identity Resource, API Resource, Users/Roles 화면 구조는 사라진다. JWT bearer handler는 token을 검증하고 Claim 목록으로 `ClaimsIdentity`와 `ClaimsPrincipal`을 만든다. API는 최종 Claim만 소비한다.

## 자주 하는 오해

### Resource의 UserClaims에 value를 저장한다고 생각한다

Resource에는 type 이름만 저장한다. 실제 value는 현재 User와 Profile Service가 제공한다.

### UserClaims에 저장하면 자동으로 모든 token에 들어간다고 생각한다

저장과 발급은 별개다. Profile Service와 요청된 scope/resource의 filtering 정책이 해당 Claim을 발급해야 한다.

### Identity Resource와 API Resource가 같은 token을 꾸민다고 생각한다

둘 다 사용자 Claim type을 선택하지만 목적이 다르다. Identity Resource는 identity 정보 공개를 모델링하고, API Resource/API Scope는 API access token에 필요한 정보를 모델링한다.

### `role`이라는 이름 자체가 권한을 부여한다고 생각한다

Claim은 데이터다. API가 해당 type을 `RoleClaimType`으로 지정하거나 authorization policy에서 검사해야 권한 판단에 사용된다.

### User가 Identity Resource라고 생각한다

User는 subject/resource owner다. Identity Resource는 그 User에 관한 요청 가능한 정보 묶음이다.

## 설계 체크리스트

- Claim type과 value를 명확히 구분했는가?
- User의 실제 데이터와 Resource의 type 선택 목록을 구분했는가?
- 각 Claim이 identity token/userinfo와 access token 중 어디에 필요한지 결정했는가?
- Claim value의 source와 Profile Service projection 규칙이 명확한가?
- 같은 type에 여러 value가 있을 때 API가 올바르게 처리하는가?
- raw user claim과 managed role assignment를 중복 사용하고 있지 않은가?
- 공유 IdP라면 제품별 claim type namespace를 사용하는가?
- API의 `RoleClaimType`이 발급된 claim type과 정확히 일치하는가?
- token을 decode해 `aud`, `scope`, claim type/value를 실제로 확인했는가?

## 정리

IdentityServer의 여러 화면에서 `UserClaims`라는 이름이 반복되지만 의미는 두 가지로 나뉜다.

```text
User의 UserClaims
  = 현재 사용자에게 속한 실제 type/value 데이터

Resource의 UserClaims
  = Profile Service에 요청할 claim type 이름 목록
```

Identity Resource는 사용자 자체가 아니라 요청 가능한 identity 정보 묶음이다. API Resource는 보호할 API다. Profile Service는 Resource가 선택한 type과 현재 User의 실제 value를 연결해 token에 발급한다.

이 구조를 이해하면 `role`도 자연스럽게 설명된다. `role`은 문자열 Claim type이지만 IdP의 role projection과 ASP.NET Core의 `RoleClaimType` 설정이 그 Claim을 role 인가에 사용하도록 만든다. 공유 IdP에서는 `omni_role`처럼 namespace가 분리된 type을 사용하는 편이 안전하다.

## Ref

- Duende IdentityServer terminology: <https://docs.duendesoftware.com/identityserver/overview/terminology/>
- Duende IdentityServer claims: <https://docs.duendesoftware.com/identityserver/fundamentals/claims/>
- Duende IdentityServer resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/>
- Duende IdentityServer identity resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/identity/>
- Duende IdentityServer API resources: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/api-resources/>
- Duende IdentityServer API scopes: <https://docs.duendesoftware.com/identityserver/fundamentals/resources/api-scopes/>
- Duende IdentityServer Profile Service: <https://docs.duendesoftware.com/identityserver/reference/v8/services/profile-service/>
- Duende IdentityServer stores: <https://docs.duendesoftware.com/identityserver/reference/v7/stores/>
- ASP.NET Core role-based authorization: <https://learn.microsoft.com/en-us/aspnet/core/security/authorization/roles>
- `ClaimsIdentity.RoleClaimType`: <https://learn.microsoft.com/en-us/dotnet/api/system.security.claims.claimsidentity.roleclaimtype>
- `TokenValidationParameters.RoleClaimType`: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.identitymodel.tokens.tokenvalidationparameters.roleclaimtype>
- `ClaimsPrincipal.IsInRole`: <https://learn.microsoft.com/en-us/dotnet/api/system.security.claims.claimsprincipal.isinrole>
- JSON Web Token, RFC 7519: <https://www.rfc-editor.org/rfc/rfc7519>
- OpenID Connect Core 1.0: <https://openid.net/specs/openid-connect-core-1_0.html>
- SCIM Core Schema, RFC 7643: <https://www.rfc-editor.org/rfc/rfc7643>
- SCIM Protocol, RFC 7644: <https://www.rfc-editor.org/rfc/rfc7644>
