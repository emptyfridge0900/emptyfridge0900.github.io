+++
title="SignalR Hub"
date=2025-05-10

[taxonomies]
categories = ["post"]
tags = ["C#"]
+++

Yesterday I had an interview with a SignalR question, and I did not feel satisfied with my answer.
The interviewer mentioned that there is something called a strongly typed hub, so I read Microsoft's [Learn page](https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs) to see how it describes Hub and Strongly Typed Hub.

### SignalR

First, briefly: SignalR is a library that enables real-time communication where the server and client can call each other's methods. In the past, if a server wanted to send something to a client, it had to wait for a client request and use techniques like Long Polling. These days WebSockets and Server-Sent Events are also common.

SignalR supports WebSockets, Server-Sent Events, and Long Polling, and automatically chooses the most appropriate transport supported by the server and client. WebSockets is generally the preferred transport. At a previous company, we used Hub for dashboard updates and system status updates.

### Hub

I have heard that Hub is SignalR's high-level API. I am not sure whether there is another common way to use SignalR besides Hub. Even the [SignalR API docs](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.signalr) are mostly about Hub, and the tutorials are all Hub tutorials.

The official page makes Hub usage easy to learn through examples. Hub is an abstract class, so you cannot use it directly; you inherit from it. The central property is `Clients`, which lets you communicate with all connected clients or selected clients. In my memory, we often used `All` and `Group` at my previous company.

One interview question was how to communicate with a specific client. I did not answer clearly at the time. To send to one specific connection, use `Clients.Client(connectionId).SendAsync(...)`. That value is not a user id; it is a SignalR connection id. To send based on authenticated user, use `Clients.User(userId)`. To send by group, use `Clients.Group(groupName)`.

### Strongly Typed Hub

When using `SendAsync`, the client method name is written as a string, and it is easy to mistype the method name. The compiler cannot catch that. If the name does not match the actual client handler, the call will not be handled as expected.

There is a way to enforce client method names and signatures in the Hub: strongly typed hubs. If you specify a client contract interface with `Hub<TClient>`, you can call methods like `Clients.All.ReceiveMessage(...)` and get compile-time checking.

### Ref

- SignalR overview: <https://learn.microsoft.com/en-us/aspnet/core/signalr/introduction>
- SignalR hubs: <https://learn.microsoft.com/en-us/aspnet/core/signalr/hubs>
