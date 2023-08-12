+++
title="Work Queues"
date=2023-08-12

[taxonomies]
categories = ["post"]
tags = ["RabbitMQ","Event bus"]
+++

<https://www.rabbitmq.com/tutorials/tutorial-two-dotnet.html>

### 1. Preparation

shell
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

shell
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

One of the advantages of using a Task Queue is the ability to easily parallelise work. If we are building up a backlog of work, we can just add more workers and that way, scale easily.
Task Queue의 장점은 병렬작업을 할수 있다는 것. 처리해야할 주문을 담당하는 일을 구축하려면 그냥 작업자를 추가하면 쉽게 확장가능
NewTask 객체 하나와 Worker 객체를 두개 만들어서 돌려보자. 둘다 queue로부터 메세지를 받게 된다

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
RabbitMQ는 다음 consumer에게 메세지를 차례대로 전달한다. 모든 consumer가 균등한 양의 메세지를 받게된다.
이런방식을 Round robin이라고 한다

### 3. Message acknowledgment

만약 어떤 consumer가 작업을 하다가 작업을 마치지 못하고 죽어버린다면? RabbitMQ는 기본적으로 consumer에게 메시지를 보내고 나면 메세지를 지워버린다. 
이런경우 consumer가 작업하다 죽어버리면 우리는 메세지를 잃어버리게된다.
메세지를 잃지 않기 위해 RabbitMQ는 message acknowlegement를 지원한다. Mesaage acknowlegement는 consumer가 RabittMQ에게 메세지를 받았다는 신호를 보내는 것이다.
메세지 수신 신호를 받으면 RabbitMQ는 메세지를 삭제하게된다.
Consumer가 ack를 보내지 않고 죽어버리게 된다면 RabbitMQ는 메세지가 처리되지 않은것으로 간주해 다시 queue에 메세지를 쌓는다. 이때 다른 consumer가 있다면 queue에 있는 메세지를 꺼내볼것이다. 

Acknowledgement timeout 기본값은 30분

Manual message acknowledgments이 기본으로 켜져있다. 앞선 예제에서는 autoAck에 true값을 주어서 명시적으로 켰다.
이번에는 끄고 수동으로 ack신호를 보내보자.

Worker.cs

BasicAck를 추가하고 autoAck:false로 만들자
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
Acknowledgement는 반드시 같은 channel로 보내져야한다.

### 4. Message durability

consumer가 멈추면 다른 worker가 메세지를 받아줄수 있다는걸 배웠다. 그런데 rabbitMQ가 멈춰버리면 작업하던거 날아간다.
메세지와 queue에 durable이라는 플레그를 주면 rabbitMQ가 멈춰도 하던작업을 기억한다.

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
이미 hello라는 이름의 queue가 있으니까 task_queue이름으로 만들어주자. 기존에 있던 queue의 파라미터를 재정의 하는것은 불가하다자.

### 5. Fair Dispatch

한쪽에만 무거운 작업이 분배되고 다른 한쪽은 가벼운 작업이 분배될 때가 있다. 하지만 RabbitMQ는 그런거 상관 안하고 똑같은 갯수의 메세지를 준다.

이런 행위를 바꾸려면 BasicQos 메소드에 prefetchCount=1 값을 설정하면 된다. 이것은 RabbitMQ한테 작업자에게 한 번에 둘 이상의 메시지를 제공하지 않도록 지시한다.
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