+++
title="Hello World"
date=2023-08-01

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ","Event bus"]
+++

<https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html>


# 준비

## 1. RabbitMQ 설치
로컬에 직접 설치하기보다는 Docker로 띄우는 방식을 선호한다.

[Docker 이미지로 설치하기](https://hub.docker.com/_/rabbitmq)
```
docker run -d
--hostname hello-world
--name test-rabbit
-p 7777:5672
-p 17777:15672
-e RABBITMQ_DEFAULT_USER=test
-e RABBITMQ_DEFAULT_PASS=test123
rabbitmq:3-management
```


## 2. Sending client

메시지를 보내는 console app을 만든다.

```ps1
dotnet new console --name Send
mv Send/Program.cs Send/Send.cs
cd Send
dotnet add package RabbitMQ.Client
```

```cs
using System.Text;
using RabbitMQ.Client;

var factory = new ConnectionFactory { HostName = "localhost", UserName="test", Password ="test123",Port=7777  };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "hello",
                     durable: false,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

const string message = "Hello World!";
var body = Encoding.UTF8.GetBytes(message);

channel.BasicPublish(exchange: string.Empty,
                     routingKey: "hello",
                     basicProperties: null,
                     body: body);
Console.WriteLine($" [x] Sent {message}");

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```



## 3. Receiving client

이번에는 queue에서 메시지를 받아 출력하는 receiver를 만든다.

```ps1
dotnet new console --name Receive
mv Receive/Program.cs Receive/Receive.cs
cd Receive
dotnet add package RabbitMQ.Client
```

```cs
using System.Text;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

var factory = new ConnectionFactory { HostName = "localhost", UserName="test", Password ="test123",Port=7777  };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "hello",
                     durable: false,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

 Console.WriteLine(" [*] Waiting for messages.");

var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($" [x] Received {message}");
};
channel.BasicConsume(queue: "hello",
                     autoAck: true,
                     consumer: consumer);

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```


# Management Plugin

`docker run`을 할 때 `-p 17777:15672` 옵션을 넣었다. `15672`는 RabbitMQ management plugin의 HTTP 포트다.
브라우저에서 <http://localhost:17777/> 로 접속하면 관리 페이지를 볼 수 있다.

## Sender, Receiver 실행 전

sender와 receiver client를 실행하기 전에는 queue가 비어 있다.

![](/images/rabbitmq/hello-world/3.png)

## Sender 실행 후

Sender client를 실행하면 queue 상태가 바뀐다. `hello` queue에 메시지가 들어간 것을 볼 수 있다.

![](/images/rabbitmq/hello-world/7.png)

## Receiver 실행 후

Sender를 종료하고 Receiver를 실행한 뒤 관리 페이지를 다시 보면 queue가 비어서 0이 되어 있다. Receiver가 메시지를 가져가 처리했기 때문이다.

![](/images/rabbitmq/hello-world/8.png)
