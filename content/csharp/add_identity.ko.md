+++
title="AddIdentity"
date=2025-06-14

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

`AddIdentityCore`와 `AddIdentity`가 무엇이 다른지 궁금한 사람은 [나뿐만이 아니었다](https://stackoverflow.com/questions/55361533/addidentity-vs-addidentitycore).

## [AddIdentityCore](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Extensions.Core/src/IdentityServiceCollectionExtensions.cs)

```cs
services.AddOptions().AddLogging();

// Services used by identity
services.TryAddScoped<IUserValidator<TUser>, UserValidator<TUser>>();
services.TryAddScoped<IPasswordValidator<TUser>, PasswordValidator<TUser>>();
services.TryAddScoped<IPasswordHasher<TUser>, PasswordHasher<TUser>>();
services.TryAddScoped<ILookupNormalizer, UpperInvariantLookupNormalizer>();
services.TryAddScoped<IUserConfirmation<TUser>, DefaultUserConfirmation<TUser>>();
// No interface for the error describer so we can add errors without rev'ing the interface
services.TryAddScoped<IdentityErrorDescriber>();
services.TryAddScoped<IUserClaimsPrincipalFactory<TUser>, UserClaimsPrincipalFactory<TUser>>();
services.TryAddScoped<UserManager<TUser>>();

```
## [AddIdentity](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Core/src/IdentityServiceCollectionExtensions.cs)

```cs
services.AddAuthentication(options =>
{
...
})
.AddCookie(IdentityConstants.ApplicationScheme, o =>
{
...
})
.AddCookie(IdentityConstants.ExternalScheme, o =>
{
...
})
.AddCookie(IdentityConstants.TwoFactorRememberMeScheme, o =>
{
...
})
.AddCookie(IdentityConstants.TwoFactorUserIdScheme, o =>
{
...
});

// Hosting doesn't add IHttpContextAccessor by default
services.AddHttpContextAccessor();
// Identity services
services.TryAddScoped<IUserValidator<TUser>, UserValidator<TUser>>();
services.TryAddScoped<IPasswordValidator<TUser>, PasswordValidator<TUser>>();
services.TryAddScoped<IPasswordHasher<TUser>, PasswordHasher<TUser>>();
services.TryAddScoped<ILookupNormalizer, UpperInvariantLookupNormalizer>();
services.TryAddScoped<IRoleValidator<TRole>, RoleValidator<TRole>>();
// No interface for the error describer so we can add errors without rev'ing the interface
services.TryAddScoped<IdentityErrorDescriber>();
services.TryAddScoped<ISecurityStampValidator, SecurityStampValidator<TUser>>();
services.TryAddEnumerable(ServiceDescriptor.Singleton<IPostConfigureOptions<SecurityStampValidatorOptions>, PostConfigureSecurityStampValidatorOptions>());
services.TryAddScoped<ITwoFactorSecurityStampValidator, TwoFactorSecurityStampValidator<TUser>>();
services.TryAddScoped<IUserClaimsPrincipalFactory<TUser>, UserClaimsPrincipalFactory<TUser, TRole>>();
services.TryAddScoped<IUserConfirmation<TUser>, DefaultUserConfirmation<TUser>>();
services.TryAddScoped<UserManager<TUser>>();
services.TryAddScoped<SignInManager<TUser>>();
services.TryAddScoped<RoleManager<TRole>>();
```

`AddIdentityCore`와 `AddIdentity`의 공통점:

1. 둘 다 user management 서비스를 제공한다.
2. 둘 다 `Microsoft.Extensions.DependencyInjection` namespace에 있는 extension method다. `AddIdentity`와 `AddIdentityCore`는 둘 다 `IdentityServiceCollectionExtensions` class에 있지만 source 위치와 포함하는 서비스 범위가 다르다.
`AddIdentity`는 [Identity.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Core) 쪽에 있고,
`AddIdentityCore`는 [Identity.Extensions.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Extensions.Core) 쪽에 있다.

`AddIdentityCore`와 `AddIdentity`의 차이점은 아래와 같다.
`AddIdentityCore<TUser>`는 지정한 user type을 위한 core Identity system을 추가한다. role service는 기본으로 추가되지 않고 `.AddRoles<TRole>()`로 붙일 수 있다.
`AddIdentity<TUser, TRole>`는 user와 role type을 모두 받으며, default Identity system configuration을 추가한다. cookie 인증, `SignInManager<TUser>`, `RoleManager<TRole>` 같은 서비스가 기본으로 들어온다.
```cs

services.AddAuthentication(o =>
 {
    o.DefaultAuthenticateScheme = IdentityConstants.ApplicationScheme;
    o.DefaultChallengeScheme = IdentityConstants.ApplicationScheme;
    o.DefaultSignInScheme = IdentityConstants.ExternalScheme;
})
.AddCookie(IdentityConstants.ApplicationScheme, o =>
{
...
})
.AddCookie(IdentityConstants.ExternalScheme, o =>
{
...
})
.AddCookie(IdentityConstants.TwoFactorRememberMeScheme, o =>
{
...
})
.AddCookie(IdentityConstants.TwoFactorUserIdScheme, o =>
{
...
});


services.AddHttpContextAccessor();


services.TryAddScoped<IRoleValidator<TRole>, RoleValidator<TRole>>();
services.TryAddScoped<ISecurityStampValidator, SecurityStampValidator<TUser>>();
services.TryAddEnumerable(ServiceDescriptor.Singleton<IPostConfigureOptions<SecurityStampValidatorOptions>, PostConfigureSecurityStampValidatorOptions>());
services.TryAddScoped<IUserConfirmation<TUser>, DefaultUserConfirmation<TUser>>();
services.TryAddScoped<SignInManager<TUser>>();
services.TryAddScoped<RoleManager<TRole>>();

```

## [AddDefaultIdentity](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/UI/src/IdentityServiceCollectionUIExtensions.cs)
```cs
services.AddAuthentication(o =>
{
    o.DefaultScheme = IdentityConstants.ApplicationScheme;
    o.DefaultSignInScheme = IdentityConstants.ExternalScheme;
})
.AddIdentityCookies(o => { });

return services.AddIdentityCore<TUser>(o =>
{
    o.Stores.MaxLengthForKeys = 128;
    configureOptions?.Invoke(o);
})
.AddDefaultUI()
.AddDefaultTokenProviders();
```
`AddDefaultIdentity<TUser>`는 흔히 쓰는 Identity 구성을 한 번에 추가한다. 공식 API 설명 기준으로 default UI, token providers를 포함하고, authentication이 Identity cookie를 사용하도록 설정한다. 내부적으로는 `AddIdentityCore<TUser>` 위에 cookie 인증, [AddDefaultUI](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/UI/src/IdentityBuilderUIExtensions.cs#L32), default token providers 등을 얹는 형태로 보면 된다.
로그인 페이지를 직접 만들 시간이 없다면 `AddDefaultIdentity`를 사용해 `AddDefaultUI`의 도움을 받을 수 있다. 다만 role은 기본으로 들어가지 않으니 필요한 경우 직접 추가해야 한다.
처음부터 튜닝하고 싶으면 AddIdentityCore를 쓰고, 모든 기능이 필요하다면 AddIdentity를 쓰면 되겠다. 근데 Identity를 처음쓰는 초보자라면 AddIdentity에 있는 기능을 다 훑어보기도 벅찰거다.

## Ref

- AddIdentityCore API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionextensions.addidentitycore>
- AddIdentity API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionextensions.addidentity>
- AddDefaultIdentity API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionuiextensions.adddefaultidentity>
- ASP.NET Core Identity overview: <https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity>
