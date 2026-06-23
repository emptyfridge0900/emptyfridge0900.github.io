+++
title="SignalR Hub"
date=2025-05-10

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

어제 인터뷰를 봤는데 SignalR 관련된 질문이 있었고 내 스스로 생각하기에 좀 만족스러운 답을 하지 못했다.  
인터뷰어가 Strongly typed hub라는게 있으니 한번 찾아보라고 해서 [마소의 Learn 페이지](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs) 에서 Hub 랑 Strongly Typed Hub 가 어떻게 묘사되있는지 읽어 보았다.

### SignalR
먼저 signalR이 뭔지 간단하게만 짚어보자. SignalR은 서버와 클라이언트가 서로 method를 호출하는 것처럼 실시간 통신을 할 수 있게 해주는 라이브러리이다. 예전에는 서버에서 클라이언트에게 뭔가를 전달하려면 클라이언트로부터 request를 받기를 기다려야했고 Long Polling 같은 기술을 썼다고 들었는데, 이제는 WebSockets, Server-Sent Events 같은 기술도 많이 쓴다고 들었다. SignalR은 WebSockets, Server-Sent Events, Long Polling을 지원하고, 서버와 클라이언트가 지원하는 범위 안에서 가장 적절한 transport를 자동으로 선택한다. WebSockets가 일반적으로 preferred transport다. 예전 회사에서는 dashboard 업데이트랑 시스템 상태 업데이트할때 Hub를 썼었다.

### Hub
Hub는 SignalR의 하이레벨 api라고 들었다. SignalR을 사용하는 방법이 Hub 외에 다른 방법이 있는지는 모르겠다. [SignalR api문서](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.signalr)를 봐도 전부 Hub 이야기뿐이고 튜토리얼도 전부 Hub 튜토리얼이다. Hub 사용 방법은 홈페이지보면 예시로 쉽게 배울 수 있다. Hub는 abstract class 여서 그대로 쓸 수는 없고 상속받아서 사용해야한다. Hub에서 중심은 Clients property인데 모든 연결된 클라이언트와 통신할 수도 있고 선택적으로 통신할 수도 있다. 내 기억으로 예전 회사에서는 All 속성이랑 Group 메소드를 자주 사용했다. 어제 질문중에 어떻게 특정 클라이언트와 통신할 수 있냐는 질문이 있었는데 어제는 제대로 답을 못했다. 특정 연결 하나에 보내려면 `Clients.Client(connectionId).SendAsync(...)`를 쓴다. 여기서 값은 user id가 아니라 SignalR connection id다. 인증된 사용자 기준으로 보내려면 `Clients.User(userId)`, 그룹 기준이면 `Clients.Group(groupName)`을 사용한다.

### Strongly Typed Hub
SendAsync 함수를 쓸때 클라이언트 함수이름을 string타입으로 적어넣어야하는데 함수이름을 잘못 적을때가 있다. 이 경우 컴파일러가 잡아주지 못하고, 실제 클라이언트 handler와 이름이 맞지 않으면 호출이 기대대로 처리되지 않는다. Hub에 클라이언트 함수 이름과 signature를 강제하는 방법이 있는데 그게 strongly typed hub다. `Hub<TClient>`로 client contract interface를 지정하면 `Clients.All.ReceiveMessage(...)`처럼 method 호출 형태로 쓸 수 있어서 compile-time check를 받을 수 있다.

### Ref

- SignalR overview: <https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction>
- SignalR hubs: <https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs>
