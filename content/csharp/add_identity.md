+++
title="AddIdentity"
date=2025-06-14

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

I was not the only person wondering about the difference between `AddIdentityCore` and `AddIdentity`: [Stack Overflow discussion](https://stackoverflow.com/questions/55361533/addidentity-vs-addidentitycore)

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

Similarities between `AddIdentityCore` and `AddIdentity`:

1. Both provide user-management services.
2. Both are extension methods in the `Microsoft.Extensions.DependencyInjection` namespace. `AddIdentity` and `AddIdentityCore` both live in an `IdentityServiceCollectionExtensions` class, but their source locations and the range of services they register are different.

`AddIdentity` is under [Identity.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Core), while `AddIdentityCore` is under [Identity.Extensions.Core](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/Extensions.Core).

The differences are roughly:

`AddIdentityCore<TUser>` adds the core Identity system for the specified user type. Role services are not added by default, but can be attached with `.AddRoles<TRole>()`.

`AddIdentity<TUser, TRole>` receives both user and role types and adds the default Identity system configuration. Cookie authentication, `SignInManager<TUser>`, `RoleManager<TRole>`, and related services are registered by default.

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

`AddDefaultIdentity<TUser>` adds the commonly used Identity setup in one call. According to the official API description, it includes the default UI and token providers, and configures authentication to use the Identity cookie. Internally, it can be understood as `AddIdentityCore<TUser>` plus cookie authentication, [AddDefaultUI](https://github.com/dotnet/aspnetcore/blob/main/src/Identity/UI/src/IdentityBuilderUIExtensions.cs#L32), default token providers, and related setup.

If you do not have time to build login pages, use `AddDefaultIdentity` and let `AddDefaultUI` help. Roles are not included by default, so add them yourself if needed.

If you want to tune everything from the beginning, use `AddIdentityCore`. If you need the full default feature set, use `AddIdentity`. But if you are new to Identity, even understanding everything `AddIdentity` includes can be a lot.

## Ref

- AddIdentityCore API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionextensions.addidentitycore>
- AddIdentity API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionextensions.addidentity>
- AddDefaultIdentity API: <https://learn.microsoft.com/en-us/dotnet/api/microsoft.extensions.dependencyinjection.identityservicecollectionuiextensions.adddefaultidentity>
- ASP.NET Core Identity overview: <https://learn.microsoft.com/en-us/aspnet/core/security/authentication/identity>
