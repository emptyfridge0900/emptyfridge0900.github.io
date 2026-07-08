+++
title="Conventional Route, Attribute Route"
date=2025-04-18

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

Recently I had to create a .NET project, so I created both MVC and API projects from templates. That made me notice middleware I had ignored before. I realized I had never really studied routing; I had only followed examples. So I looked around.

According to Microsoft's documentation, a route template is used to match URL paths to actions and to generate URLs when creating links.

An action is a method in a controller that handles a request. In ASP.NET Core MVC, public controller methods are actions by default. If you do not want a method exposed as an action, add `[NonAction]`.

Controller actions are usually matched through **conventional routing** or **attribute routing**.
Conventional routing is often used in controller/view-based MVC apps. REST APIs are recommended to use attribute routing because it expresses resources and HTTP verbs clearly. Both can be used with controllers.

Personally, it has been years since I worked on an MVC project, so I have barely touched MVC routing. At work, my main job was building APIs, so I usually used attributes directly on controllers.

## Routes that come with templates

When creating projects from templates, these route setup methods appear often:

**MapControllers**
**MapControllerRoute**
**MapDefaultControllerRoute**

All of them belong to the `ControllerEndpointRouteBuilderExtensions` class.

### MapControllers

**Adds endpoints for controller actions to the IEndpointRouteBuilder without specifying any routes.**

This is the default style in API projects. It uses attribute routing.

### MapControllerRoute

**Adds endpoints for controller actions to the IEndpointRouteBuilder and specifies a route with the given name, pattern, defaults, constraints, and dataTokens.**

This is the routing style usually used in MVC projects.
`MapControllerRoute` creates a single route.

```cs
  endpoints.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

This is the default route setup when creating an MVC project from the template.
The default controller is `Home`, the default method is `Index`, and `id` is optional.

### MapDefaultControllerRoute

**Adds endpoints for controller actions to the IEndpointRouteBuilder and adds the default route {controller=Home}/{action=Index}/{id?}**

This is essentially `MapControllerRoute` with `{controller=Home}/{action=Index}/{id?}` already supplied. You can use `MapControllerRoute` instead.

`MapDefaultControllerRoute()` is a convenience method equivalent to this:

```cs
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

## Conclusion

There are many routing configuration details, but today I only looked at how these three methods differ.
If I need more routing detail later, I can read the rest of the documentation.

The web app documentation has subsections like Razor Pages, MVC, and client-side, so I am not sure why routing is discussed mainly under MVC.

The practical conclusion: when building APIs, use `MapControllers` and read attribute routing docs as needed. When building MVC apps, use `MapControllerRoute`. Done.

### Ref

- Routing to controller actions: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/routing>
- Handle requests with controllers: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/actions>
