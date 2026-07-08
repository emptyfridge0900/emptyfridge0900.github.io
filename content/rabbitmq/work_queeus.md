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

One advantage of a Task Queue is that it allows parallel work. If we need to build a system that handles a growing backlog of jobs, we can scale easily by adding more workers.

Create one `NewTask` object and run two `Worker` objects. Both receive messages from the queue.

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

Result
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
RabbitMQ delivers messages to the next consumer in order. Each consumer receives an equal number of messages.

This method is called round-robin.

### 3. Message acknowledgment

What happens if a consumer dies while processing a task and cannot finish it? By default, RabbitMQ deletes a message after sending it to a consumer.

In that case, if the consumer dies during processing, we lose the message.

To avoid losing messages, RabbitMQ supports message acknowledgment. A message acknowledgment is a signal from the consumer to RabbitMQ saying that the message was received and processed.

When RabbitMQ receives the acknowledgment, it deletes the message.

If a consumer dies without sending an ack, RabbitMQ treats the message as unprocessed and puts it back into the queue. If another consumer exists, that consumer can take the message from the queue.

The default acknowledgement timeout is 30 minutes.

Manual message acknowledgments are enabled by default. In the previous example, we explicitly enabled auto acknowledgment by setting `autoAck` to `true`.

This time, turn it off and send the ack manually.

Worker.cs

Add `BasicAck` and set `autoAck: false`.
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
The acknowledgement must be sent on the same channel.

### 4. Message durability

We learned that if a consumer stops, another worker can receive the message. But if RabbitMQ itself stops, the queued work disappears.

If we mark both the message and the queue as durable, RabbitMQ can remember the work even after it stops.

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
Because a queue named `hello` already exists, create a new one named `task_queue`. You cannot redefine the parameters of an existing queue.

### 5. Fair Dispatch

Sometimes heavy tasks are distributed to one worker while light tasks go to another. RabbitMQ does not know that difference by default; it just gives each worker the same number of messages.

To change this behavior, set `prefetchCount=1` on the `BasicQos` method. This tells RabbitMQ not to give a worker more than one message at a time.

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
