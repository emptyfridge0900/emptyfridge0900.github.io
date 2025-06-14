+++
title="AddIdentity"
date=2025-06-14

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

AddIdentityCore와 AddIdentity가 무엇이 다른지 궁굼한 사람은 [나 뿐만이 아니였다](https://stackoverflow.com/questions/55361533/addidentity-vs-addidentitycore)

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

AddIdentityCore 와 AddIdentity 같은점
1. 둘다 유저관리 서비스를 제공한다  
2. 같은 네임스페이스 IdentityServiceCollectionExtensions 에 속해있다 하지만 코드는 다른 폴더에 있는데 
AddIdentity 는 [Identity.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Core) 
AddIdentityCore 는 [Identity.Extensions.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Extensions.Core) 
에 위치해있다.

AddIdentityCore 와 AddIdentity 다른 점은 아래와 같다.  
인증 관련한 서비스들이 추가된 것을 볼 수 있다. SignInManager나 RoleManager가 기본으로 딸려나온다.
```cs

services.AddAuthentication(o =>
 {
    options.DefaultAuthenticateScheme = IdentityConstants.ApplicationScheme;
    options.DefaultChallengeScheme = IdentityConstants.ApplicationScheme;
    options.DefaultSignInScheme = IdentityConstants.ExternalScheme;
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
AddIdentityCore와 대부분 같고 AddIdentityCore와 다른점은 기본적인 cookie-based 인증과 [AddDefaultUI](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/UI/src/IdentityBuilderUIExtensions.cs#L32) 를 추가해준다는 것이다.  
로그인 페이지를 만들 시간이 없다면 AddDefaultIdentity를 사용하여 AddDefultUI의 도움을 받자. Role이 기본으로 들어가지 않으니 알아서 추가해야한다.  
처음부터 튜닝하고 싶으면 AddIdentityCore를 쓰고, 모든 기능이 필요하다면 AddIdentity를 쓰면 되겠다. 근데 Identity를 처음쓰는 초보자라면 AddIdentity에 있는 기능을 다 훑어보기도 벅찰거다.



