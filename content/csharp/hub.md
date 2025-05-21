+++
title="SignalR Hub"
date=2025-05-10

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

어제 인터뷰를 봤는데 SignalR 관련된 질문이 있었고 내 스스로 생각하기에 좀 만족스러운 답을 하지 못했다.  
인터뷰어가 Strongly typed hub라는게 있으니 한번 찾아보라고 해서 [마소의 Learn 페이지](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs?view=aspnetcore-9.0) 에서 Hub 랑 Strongly Typed Hub 가 어떻게 묘사되있는지 읽어 보았다.

### SignalR
먼저 signalR이 뭔지 간단하게만 짚어보자. SignalR은 서버 대 클라이언트 통신하는 RPC를 생성해서 컨텐츠를 전달해주는 라이브러리이다. 예전에는 서버에서 클라이언트에게 뭔가를 전달하려면 클라이언트로부터 request를 받기를 기다려야했고 long pooling 같은 기술을 썼다고 들었는데, 이제는 web socket, server side event 같은 기술도 많이 쓴다고 들었다. SignalR은 long pooling, web socket, server side event 모두를 지원한다고 한다. 어떻게 하는지는 모르겠지만 알아서 최적의 방법으로 서버와 클라이언트의 통신을 지원한다는게 마소의 설명이다. 예전 회사에서는 dashboard 업데이트랑 시스템 상태 업데이트할때 Hub를 썼었다.

### Hub
Hub는 SignalR의 하이레벨 api라고 들었다. SignalR을 사용하는 방법이 Hub 외에 다른 방법이 있는지는 모르겠다. [SignalR api문서](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.signalr?view=aspnetcore-9.0)를 봐도 전부 Hub 이야기뿐이고 튜토리얼도 전부 Hub 튜토리얼이다. Hub 사용 방법은 홈페이지보면 예시로 쉽게 배울 수 있다. Hub는 abstract class 여서 그대로 쓸 수는 없고 상속받아서 사용해야한다. Hub에서 중심은 Clients property인데 모든 연결된 클라이언트와 통신할 수도 있고 선택적으로 통신할 수도 있다. 내 기억으로 예전 회사에서는 All 속성이랑 Group 메소드를 자주 사용했다. 어제 질문중에 어떻게 특정 클라이언트와 통신할 수 있냐는 질문이 있었는데 어제는 제대로 답을 못했다. Client 메소드에 client id를 넣으면 그 클라이언트와의 연결이 되고 이제 SendAsync()로 메세지를 주고 받으면 된다.

### Strongly Typed Hub
SendAsync 함수를 쓸때 함수이름을 string타입으로 적어넣어야하는데 함수이름을 잘못 적을때가 있다. 함수이름이 틀리면 런타임 에러가 난다. Hub에 함수이름을 강제하는 방법이 있는데 그게 strongly typed hub다. 