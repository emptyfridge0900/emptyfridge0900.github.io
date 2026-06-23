+++
title="Conventional Route, Attribute Route"
date=2025-04-18

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

최근에 닷넷 프로젝트를 생성할 일이 있어서 mvc랑 api 프로젝트를template로 생성해봤는데 그동안 무시하고 지나갔던 미들웨어들이눈에 들어왔다. 생각해보니 라우팅 관련해서 공부해본적도 없고 그냥 예제 따라했던게 전부여서 인터넷을 뒤적여봤다.  


일단 마소 공식 홈페이지에서 보니 route template는 URL path를 action에 매칭하고, link를 만들 때 URL을 생성하는 데 쓰인다고 한다.
action은 controller에서 request를 처리하는 method다. ASP.NET Core MVC에서는 public controller method가 기본적으로 action이고, action으로 노출하고 싶지 않으면 `[NonAction]`을 붙일 수 있다.  
controller action은 보통 **conventional routing** 또는 **attribute routing**으로 매칭된다.  
conventional routing은 controller/view 기반 MVC 앱에서 자주 쓰이고, REST API는 resource와 HTTP verb를 명확히 표현하기 위해 attribute routing을 쓰는 것이 권장된다. 둘 다 controller에서 사용할 수 있다.  
개인적으로는 mvc 프로젝트한게 몇년전이여서 mvc 라우팅은 건드려본적이 없다.  
대신 회사에 일할때는 api만들어서 사용하는게 주된 업무여서 항상 controller에 attribute를 달아서 사용했다.

## 템플릿에 딸려나오는 route
개발하다 보면 template을 통해 많이 보게되는 Route세팅 방식이 있다  
**MapControllers**  
**MapControllerRoute**  
**MapDefaultControllerRoute**   
모두 `ControllerEndpointRouteBuilderExtensions` class에 속한 method다


### MapControllers
**Adds endpoints for controller actions to the IEndpointRouteBuilder without specifying any routes.**  
api 프로젝트를 생성하면 기본적으로 사용되는 방식, attribute-route다.

### MapControllerRoute
**Adds endpoints for controller actions to the IEndpointRouteBuilder and specifies a route with the given name, pattern, defaults, constraints, and dataTokens.**  
mvc 프로젝트를 생성하면 주로  사용되는 라우터방식
MapControllerRoute is used to create a single route.
```cs
  endpoints.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```
이게 mvc  template 사용해서 프로젝트를 만들면 기본적으로 주어지는 라우트세팅이다.
기본 컨트롤러는 home, 기본 메소드는 index, id는 optional.


### MapDefaultControllerRoute
**Adds endpoints for controller actions to the IEndpointRouteBuilder and adds the default route {controller=Home}/{action=Index}/{id?}**  
사실상 MapControllerRoute에 {controller=Home}/{action=Index}/{id?} 기본값을 넣은 것으로 이것대신 MapControllerRoute쓰면된다.
`MapDefaultControllerRoute()`는 아래 코드와 같은 convenience method다.

```cs
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

## 결론
라우터에 대한 설정은 엄청 많은데 오늘은 그냥 3가지 미들웨어가 어떻게 다른지만 읽어보았다.
라우터 설정에 대해 더 알고싶으면 나머지 내용은 나중에 읽어보면 될거 같다.
webapp 섹션에 razor pages, mvc, client-side 등등 여러 하위섹션이 있는데 왜 mvc에서만 라우팅을 다루는지 모르겠다.
결론은 api 개발을 하면 MapControllers사용하고 필요할때마다 attribute-route 사용방법 읽어보면되고, mvc하면 MapControllerRoute 사용하면 된다. 끝!

### Ref

- Routing to controller actions: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/routing>
- Handle requests with controllers: <https://learn.microsoft.com/en-us/aspnet/core/mvc/controllers/actions>
