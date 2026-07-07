+++
title="SignalR Hub"
date=2025-05-10

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

어제 인터뷰를 봤는데 SignalR 관련된 질문이 있었고 내 스스로 생각하기에 좀 만족스러운 답을 하지 못했다.
인터뷰어가 strongly typed hub라는 것이 있으니 한번 찾아보라고 해서 [Microsoft Learn 페이지](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs)에서 Hub와 Strongly Typed Hub가 어떻게 설명되어 있는지 읽어 보았다.

### SignalR
먼저 SignalR이 무엇인지 간단히 짚고 넘어가자. SignalR은 서버와 클라이언트가 서로 method를 호출하는 것처럼 실시간 통신을 할 수 있게 해주는 라이브러리다. 예전에는 서버가 클라이언트에게 무언가를 전달하려면 클라이언트의 request를 기다리거나 Long Polling 같은 기법을 사용해야 했다. 지금은 WebSockets, Server-Sent Events 같은 기술도 많이 사용한다. SignalR은 WebSockets, Server-Sent Events, Long Polling을 지원하고, 서버와 클라이언트가 지원하는 범위 안에서 가장 적절한 transport를 자동으로 선택한다. 일반적으로 WebSockets가 preferred transport다. 예전 회사에서는 dashboard 업데이트와 시스템 상태 업데이트에 Hub를 사용했었다.

### Hub
Hub는 SignalR의 high-level API다. SignalR을 Hub 외의 방식으로 사용하는 경우도 있겠지만, [SignalR API 문서](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.signalr)를 봐도 대부분 Hub 중심이고 튜토리얼도 Hub를 기준으로 설명한다. Hub 사용 방법은 공식 문서의 예시를 보면 쉽게 배울 수 있다. Hub는 abstract class라서 그대로 사용할 수 없고 상속해서 사용해야 한다. Hub에서 중심이 되는 것은 `Clients` property다. 이 property를 통해 연결된 모든 클라이언트와 통신할 수도 있고, 특정 클라이언트나 그룹을 골라 통신할 수도 있다. 내 기억으로 예전 회사에서는 `All` property와 `Group` method를 자주 사용했다.

인터뷰 중에 특정 클라이언트와 어떻게 통신할 수 있느냐는 질문이 있었는데, 당시에는 제대로 답하지 못했다. 특정 connection 하나에 보내려면 `Clients.Client(connectionId).SendAsync(...)`를 사용한다. 여기서 값은 user id가 아니라 SignalR connection id다. 인증된 사용자 기준으로 보내려면 `Clients.User(userId)`, 그룹 기준이면 `Clients.Group(groupName)`을 사용한다.

### Strongly Typed Hub
`SendAsync`를 사용할 때는 클라이언트 함수 이름을 string으로 적어야 한다. 이때 함수 이름을 잘못 적어도 컴파일러가 잡아주지 못한다. 실제 클라이언트 handler 이름과 맞지 않으면 호출이 기대대로 처리되지 않는다.

Hub에서 클라이언트 함수 이름과 signature를 강제하는 방법이 있는데, 이것이 strongly typed hub다. `Hub<TClient>`로 client contract interface를 지정하면 `Clients.All.ReceiveMessage(...)`처럼 method 호출 형태로 사용할 수 있어서 compile-time check를 받을 수 있다.

### Ref

- SignalR overview: <https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction>
- SignalR hubs: <https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs>
