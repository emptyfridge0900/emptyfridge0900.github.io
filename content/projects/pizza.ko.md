+++
title="Pizza Shop"
date=2020-12-05

[taxonomies]
categories = ["Side Project"]
tags = ["project", "C#","SQL","EF core","Blazor","ASP.NET","Front-End","Back-End"]
+++

# GitHub Repository
[여기서 보기](https://github.com/emptyfridge0900/pizza)


# 프로젝트
가상의 피자 가게에서 피자와 사이드 메뉴를 주문하는 프로젝트다. 요즘 OAuth2가 여기저기 많이 쓰이기 때문에, 이 프로젝트에서는 IdentityServer4를 사용해 보았다. 물론 실제 피자를 사는 데 로그인까지 필요하지는 않겠지만, 인증/인가 흐름을 연습하기 위한 선택이었다.

# 동기
지금까지 backend, frontend, SQL을 각각 공부해 왔지만, 이들을 통합해서 하나의 제품처럼 배포해 본 경험은 없었다. 완성도는 부족하더라도, 이 프로젝트를 통해 실제 제품을 만들어 보는 경험을 얻고 싶었다.

또 이 프로젝트에서는 이전에 사용해 보지 않았던 기술들을 일부러 사용했다. Docker, Blazor, Bootstrap, PostgreSQL에 대해 많이 배울 수 있었다.

# GIF
![](/images/login.gif)
![](/images/history.gif)
![](/images/add.gif)

# 사용한 기술/framework
- [ASP .NET Core 5.0 Api](https://docs.microsoft.com/en-us/aspnet/core/web-api/?view=aspnetcore-5.0)
- [ASP .NET Core 5.0 Blazor(server)](https://docs.microsoft.com/en-us/aspnet/core/blazor/?view=aspnetcore-5.0)
- [Identity Server 4](https://identityserver4.readthedocs.io/en/latest/)
- [Entity Framework Core](https://docs.microsoft.com/en-us/ef/core/)
- [Postgres 12.5](https://www.postgresql.org/docs/12/index.html)
- [Bootstrap 4](https://getbootstrap.com/)
- [Docker](https://docs.docker.com/get-started/overview/)
### 호스팅 서비스
- [Heroku](https://www.heroku.com/)


# 기능

API
- 피자 목록 조회
- 사이드 메뉴 목록 조회
- 주문 생성
- 주문 내역 조회
- 주문 상태 추적

CLIENT

- 피자 정보 표시
- 사이드 메뉴 정보 표시
- 장바구니에 항목 추가
- 세금과 결제 금액 계산
- Checkout
- Login


# API Reference
## GET methods
### **api/pizza/type**
- 피자 종류 조회


### **api/pizza/size**
- 피자 크기 조회


### **api/pizaz/topping**
- 피자 토핑 조회


### **api/pizza/sides**
- 사이드 메뉴 조회


### **api/Pizza/Customers/{id}/Orders**
- customer id로 주문 내역 조회
`parameter: Guid`


### **api/Pizza/Orders/{orderId}**
- orderId로 주문 상세 조회
`parameter: Guid`




## POST method
### **api/Pizza**
- 주문 생성


> body
```
{
    "Customer":{
        "CustomerId":"7d9ecead-eca1-48b2-8366-784fb1c17c16",
        "FirstName":"Bob",
        "LastName":"Smith",
        "Email":"BobSmith@email.com",
        "PhoneNumber":"519-666-7777"
    },
    "Pizzas":[
        {
            "Qty":2,
            "TypeId":1,
            "SizeId":1,
            "Toppings":[1,2,3]

        }
    ],
    "Sides":[
        {
            "Qty":2,
            "SideId":1

        }
    ]
}
```
