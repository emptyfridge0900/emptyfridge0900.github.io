+++
title="Hello World"
date=2023-08-01

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ","Event bus"]
+++

<https://www.rabbitmq.com/tutorials/tutorial-one-dotnet.html>


# setup

## 1. RabbitMQ 설치
나는 로컬에 설치하기 보다는 docker를 사용해서 설치하는걸 선호한다

[docker 이미지로 설치하기](https://hub.docker.com/_/rabbitmq)
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


## 2. Sedning client
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



## 3. Receving client
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
docker run을 할때 -p 17777:15672 옵션을 넣었던걸 기억하는가? 15672는 관리페이지 플러그인 포트다. 
<http://localhost:17777/> 를 통하여 관리페이지에 접속할수 있다.
## Sender, Receiver 실행 시키기 전
sender, receiver 클라이언트를 실행시키기 전에는  queue가 비어있는 것을 볼수 있다
![](/images/rabbitmq/hello-world/3.png)
## Sender 실행 시킨 후
Sender 클라이언트를 실행시킨후에는 queue에 변화가 있다.
Queue에 메세지가 들어가 있는것을 볼수 있다
![](/images/rabbitmq/hello-world/7.png)
## Receiver 실행 시킨 후
Sender를 종료시키고 Receiver를 실행시킨후에 관리페이지를 보면 Queue가 다시 비어서 0이 되어있다.
![](/images/rabbitmq/hello-world/8.png)