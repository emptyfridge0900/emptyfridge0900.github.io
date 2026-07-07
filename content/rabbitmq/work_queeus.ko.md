+++
title="Work Queues"
date=2023-08-12

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ","Event bus"]
+++

<https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html>

### 1. 준비

작업을 queue에 넣는 producer 역할의 `NewTask` console app을 만든다.

```ps1
dotnet new console --name NewTask
mv NewTask/Program.cs NewTask/NewTask.cs
cd NewTask
dotnet add package RabbitMQ.Client
```

NewTask.cs
```cs,hl_lines=14 26-29
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

var message = GetMessage(args);
var body = Encoding.UTF8.GetBytes(message);

channel.BasicPublish(exchange: string.Empty,
                     routingKey: "hello",
                     basicProperties: null,
                     body: body);
Console.WriteLine($" [x] Sent {message}");

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();

static string GetMessage(string[] args)
{
    return ((args.Length > 0) ? string.Join(" ", args) : "Hello World!");
}
```

이번에는 queue에서 작업을 꺼내 처리하는 `Worker` console app을 만든다.

```ps1
dotnet new console --name Worker
mv Worker/Program.cs Worker/Worker.cs
cd Worker
dotnet add package RabbitMQ.Client
```

Worker.cs
```cs,hl_lines=24-27
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

    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");
};
channel.BasicConsume(queue: "hello",
                     autoAck: true,
                     consumer: consumer);

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```



### 2. Round-robin dispatching

Task Queue를 쓰는 장점 중 하나는 작업을 쉽게 병렬 처리할 수 있다는 점이다. 처리해야 할 작업이 쌓이면 worker를 더 띄우면 된다. 이렇게 하면 같은 queue를 바라보는 worker 수만 늘려도 처리량을 쉽게 확장할 수 있다.

`NewTask` 하나와 `Worker` 두 개를 실행해 보자. 두 worker가 같은 queue에서 메시지를 나눠 받는다.

NewTask
```ps1
dotnet run "First message."
dotnet run "Second message.."
dotnet run "Third message..."
dotnet run "Fourth message...."
dotnet run "Fifth message....."
```

Worker
```ps1
dotnet run
```

Worker
```ps1
dotnet run
```

결과
```ps1
Press [enter] to exit.
[x] Received First message.
[x] Done
[x] Received Third message...
[x] Done
[x] Received Fifth message.....
[x] Done
```

```ps1
Press [enter] to exit.
[x] Received Second message..
[x] Done
[x] Received Fourth message....
[x] Done
```

RabbitMQ는 메시지를 다음 consumer에게 차례대로 전달한다. 그래서 모든 consumer가 비슷한 개수의 메시지를 받는다. 이런 방식을 round-robin dispatching이라고 한다.

### 3. Message acknowledgment

consumer가 작업을 처리하다가 끝내지 못하고 죽으면 어떻게 될까? RabbitMQ는 `autoAck=true`일 때 consumer에게 메시지를 보내는 순간 메시지를 처리된 것으로 간주한다. 이 경우 consumer가 중간에 죽으면 메시지를 잃는다.

메시지를 잃지 않기 위해 RabbitMQ는 message acknowledgment를 지원한다. acknowledgment는 consumer가 RabbitMQ에게 "이 메시지 처리를 끝냈다"라고 알려주는 신호다. RabbitMQ는 ack를 받은 뒤에야 메시지를 queue에서 제거한다.

consumer가 ack를 보내기 전에 죽으면 RabbitMQ는 메시지가 처리되지 않았다고 보고 다시 queue에 넣는다. 다른 consumer가 있다면 그 consumer가 해당 메시지를 다시 가져가 처리한다.

acknowledgment timeout 기본값은 30분이다.

RabbitMQ tutorial 기준으로는 manual acknowledgment를 쓰는 것이 안전한 방식이다. 앞선 예제에서는 `autoAck: true`를 줘서 자동 ack를 켰다. 이번에는 자동 ack를 끄고, 작업이 끝난 뒤 직접 ack를 보내보자.

Worker.cs

`BasicAck`를 추가하고 `autoAck: false`로 바꾼다.
```cs,hl_lines=28-29 32
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
    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");

    // here channel could also be accessed as ((EventingBasicConsumer)sender).Model
    channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
};
channel.BasicConsume(queue: "hello",
                     autoAck: false,
                     consumer: consumer);

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```

acknowledgment는 반드시 메시지를 받은 것과 같은 channel로 보내야 한다.

### 4. Message durability

consumer가 죽으면 다른 worker가 메시지를 이어서 받을 수 있다는 것을 봤다. 그런데 RabbitMQ broker 자체가 멈추면 어떻게 될까? queue와 message를 durable/persistent로 설정하지 않으면 broker 재시작 시 작업이 사라질 수 있다.

queue에는 `durable: true`를 주고, message에는 persistent property를 설정한다.

NewTask.cs
```cs,hl_lines=8-9 17-18 21-22
using System.Text;
using RabbitMQ.Client;

var factory = new ConnectionFactory { HostName = "localhost", UserName="test", Password ="test123",Port=7777  };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "task_queue",
                     durable: true,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

var message = GetMessage(args);
var body = Encoding.UTF8.GetBytes(message);

var properties = channel.CreateBasicProperties();
properties.Persistent = true;

channel.BasicPublish(exchange: string.Empty,
                     routingKey: "task_queue",
                     basicProperties: properties,
                     body: body);
Console.WriteLine($" [x] Sent {message}");

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();

static string GetMessage(string[] args)
{
    return ((args.Length > 0) ? string.Join(" ", args) : "Hello World!");
}
```

Worker.cs
```cs,hl_lines=9-10 31
using System.Text;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

var factory = new ConnectionFactory { HostName = "localhost", UserName="test", Password ="test123",Port=7777  };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "task_queue",
                     durable: true,
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
    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");

    // here channel could also be accessed as ((EventingBasicConsumer)sender).Model
    channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
};
channel.BasicConsume(queue: "task_queue",
                     autoAck: false,
                     consumer: consumer);

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```

이미 `hello`라는 queue가 `durable: false`로 존재하므로, 여기서는 `task_queue`라는 새 이름을 쓴다. RabbitMQ에서는 이미 존재하는 queue의 parameter를 다른 값으로 다시 선언할 수 없다. 바꾸려면 기존 queue를 삭제하고 새로 만들어야 한다.

### 5. Fair Dispatch

round-robin은 메시지 개수만 균등하게 나눈다. 그래서 한쪽 worker에는 오래 걸리는 작업이 몰리고, 다른 worker에는 가벼운 작업만 가는 상황이 생길 수 있다. RabbitMQ는 기본적으로 작업의 무게를 보지 않고 같은 개수의 메시지를 나눠 준다.

이 동작을 바꾸려면 `BasicQos`에서 `prefetchCount: 1`을 설정한다. 이 설정은 RabbitMQ에게 "worker 하나가 ack하지 않은 메시지를 이미 들고 있다면, 그 worker에게 새 메시지를 더 주지 말라"고 지시한다.

Worker.cs
```cs,hl_lines=15
using System.Text;
using RabbitMQ.Client;
using RabbitMQ.Client.Events;

var factory = new ConnectionFactory { HostName = "localhost", UserName="test", Password ="test123",Port=7777  };
using var connection = factory.CreateConnection();
using var channel = connection.CreateModel();

channel.QueueDeclare(queue: "task_queue",
                     durable: true,
                     exclusive: false,
                     autoDelete: false,
                     arguments: null);

channel.BasicQos(prefetchSize: 0, prefetchCount: 1, global: false);

Console.WriteLine(" [*] Waiting for messages.");

var consumer = new EventingBasicConsumer(channel);
consumer.Received += (model, ea) =>
{
    var body = ea.Body.ToArray();
    var message = Encoding.UTF8.GetString(body);
    Console.WriteLine($" [x] Received {message}");
    int dots = message.Split('.').Length - 1;
    Thread.Sleep(dots * 1000);

    Console.WriteLine(" [x] Done");

    // here channel could also be accessed as ((EventingBasicConsumer)sender).Model
    channel.BasicAck(deliveryTag: ea.DeliveryTag, multiple: false);
};
channel.BasicConsume(queue: "task_queue",
                     autoAck: false,
                     consumer: consumer);

Console.WriteLine(" Press [enter] to exit.");
Console.ReadLine();
```
