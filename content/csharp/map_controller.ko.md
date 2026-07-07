+++
title="Conventional Route와 Attribute Route"
date=2025-04-18

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

최근에 .NET 프로젝트를 생성할 일이 있어서 MVC와 API 프로젝트를 template로 만들어 보았다. 그동안 무심코 지나쳤던 middleware들이 눈에 들어왔다. 생각해 보니 routing을 제대로 공부해 본 적은 없고 예제만 따라 했던 것이 전부라서 내용을 찾아보았다.


Microsoft 공식 문서를 보면 route template은 URL path를 action에 매칭하고, link를 만들 때 URL을 생성하는 데 사용된다고 한다.
action은 controller에서 request를 처리하는 method다. ASP.NET Core MVC에서는 public controller method가 기본적으로 action이고, action으로 노출하고 싶지 않으면 `[NonAction]`을 붙일 수 있다.

controller action은 보통 **conventional routing** 또는 **attribute routing**으로 매칭된다.
conventional routing은 controller/view 기반 MVC 앱에서 자주 쓰이고, REST API에서는 resource와 HTTP verb를 명확히 표현하기 위해 attribute routing을 쓰는 것이 권장된다. 둘 다 controller에서 사용할 수 있다.

개인적으로는 MVC 프로젝트를 한 지 몇 년이 지나 MVC routing을 직접 건드려 본 적은 거의 없다. 대신 회사에서는 API를 만들어 사용하는 일이 주된 업무였기 때문에 항상 controller에 attribute를 달아서 사용했다.

## 템플릿에 딸려나오는 route
개발하다 보면 template에서 자주 보게 되는 route 설정 방식이 있다.
**MapControllers**
**MapControllerRoute**
**MapDefaultControllerRoute**
모두 `ControllerEndpointRouteBuilderExtensions` class에 속한 method다.


### MapControllers
**Adds endpoints for controller actions to the IEndpointRouteBuilder without specifying any routes.**

API 프로젝트를 생성하면 기본적으로 사용되는 방식이다. route 자체는 controller/action의 attribute로 지정하는 attribute routing 방식이다.

### MapControllerRoute
**Adds endpoints for controller actions to the IEndpointRouteBuilder and specifies a route with the given name, pattern, defaults, constraints, and dataTokens.**

MVC 프로젝트를 생성하면 주로 사용되는 routing 방식이다. `MapControllerRoute`는 하나의 conventional route를 만든다.
```cs
  endpoints.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```
MVC template로 프로젝트를 만들면 기본적으로 주어지는 route 설정이다.
기본 controller는 `Home`, 기본 action은 `Index`, `id`는 optional이다.


### MapDefaultControllerRoute
**Adds endpoints for controller actions to the IEndpointRouteBuilder and adds the default route {controller=Home}/{action=Index}/{id?}**

사실상 `MapControllerRoute`에 `{controller=Home}/{action=Index}/{id?}` 기본값을 넣은 convenience method다. 필요하면 이것 대신 `MapControllerRoute`를 직접 사용하면 된다.
`MapDefaultControllerRoute()`는 아래 코드와 같은 convenience method다.

```cs
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

## 결론
routing 설정은 훨씬 더 많지만, 오늘은 template에서 자주 보이는 세 method가 어떻게 다른지만 읽어보았다.
더 깊게 알고 싶으면 나머지 routing 문서를 나중에 읽어보면 될 것 같다.

결론은 단순하다. API 개발을 하면 `MapControllers`를 사용하고 필요할 때마다 attribute routing 사용법을 확인하면 된다. MVC라면 `MapControllerRoute`를 사용하면 된다. 끝.

### Ref

- Routing to controller actions: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/routing>
- Handle requests with controllers: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/actions>
