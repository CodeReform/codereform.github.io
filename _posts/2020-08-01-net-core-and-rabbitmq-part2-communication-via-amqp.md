---
layout: post
title: .NET Core and RabbitMQ Part 2 - Communication via AMQP
date: 2020-08-01 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- ".NET Core"
- C#
tags:
- "RabbitMQ"
- C#
- AMQP
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2020/08/01/net-core-and-rabbitmq-part2-communication-via-amqp"
---
![](https://miro.medium.com/1*auDrXWq5X6uWWbOfayXe4Q.jpeg)

# .NET Core and RabbitMQ Part 2 - Communication via AMQP

In [part 1](https://medium.com/@giorgos.dyrrahitis/net-core-and-rabbitmq-5f3c76f39de6) I demonstrated how to create a simple consumer and producer using ASP.NET Core Hosted Services. In this part, I will cover everything that happens under the wraps, the communication between the client and the server, the connection, the message publishing and consuming.

## AMQP

[AMQP](https://www.amqp.org/) is the protocol that RabbitMQ uses to communicate with client applications. Aside from the fact that [AMQP](https://www.amqp.org/) defines the wire protocol for RabbitMQ to communicate, it also provides some logical model that comprises of classes and methods which RabbitMQ adopts in its core functionality.

In its core, AMQP defines three components that are very crucial in implementing a message base architecture.

1. The exchange, which routes messages to queues. By reading the message properties, the exchange can determine to which queue should route each message.

2. The queue, which provides a FIFO data structure that stores messages either on memory or disk (persistent).

3. The binding, which defines a relationship between a queue and an exchange. When a message is published to that exchange, its routing key, or other message properties (depends the routing strategy that's applied) are evaluated against that binding. The routing key might be the name of the queue or it might be an arbitrary string that can be matched by pattern matching rules. When it's successfully evaluated, the exchange can determine to which queues it should route its messages.

With AMQP, communication is bi-directional, which means the client talks to the server but also the server can talk directly to the client. These conversations are RPC, as per AMQP specification and are issued whenever the client talks to the server, for example to connect or publish messages or when the server talks to the client, for example when the client consumes messages.

### Connection

Let's look how the first conversation between the client and the server looks like. In the previous post, I'm connecting to the server by calling the [CreateConnection](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/type-RabbitMQ.Client.IConnectionFactory.html#method-M:RabbitMQ.Client.IConnectionFactory.CreateConnection) method on the connection factory object. This kickstarted the following

1. Client sends a protocol header frame

2. Server responds with a Connection.Start command

3. Client responds with a Connection.StartOk command

![](https://miro.medium.com/0*pHdBAk8KS-ycz8sv)

Few things to notice from the above.

The conversation kickstarts by a protocol header frame. This is a special frame (more on them in a while) that is used only during connection.

Next, server issues a command, the Connection.Start. AMQP is a protocol which defines a unique dialect that is usually more or less adopted by implementations of that protocol, in this case RabbitMQ. Its aim is to make a common language between the client and the server. The commands define a class and a method, just like in a OOP language, like C#. The class, groups functionality in a single place and the method executes some task. In the example above, the Connection is the class and Start is the method of that class. The Connection.StartOk is merely a response to the command issued earlier. A command usually carries something, like arguments or some other data. Mostly, these data are encoded and the command is encapsulated in a data structure called _frame_.

### Channels

Connecting to RabbitMQ is not enough if you want your application to be up and running with the broker. It is required to open at least one channel for the connection. Be advised there's no issue if you need more than one channels open on a single connection, this is totally doable and it's called multiplexing, however for every open channel, resources are allocated on the server side, so be careful with how many channels you have open as they are costly, resource-wise and might degrade your server's performance.

## Frames

A frame is a formal data structure in the AMQP specification which carries information and it's meaningful for a RabbitMQ server or client. It can contain metadata or actual content. Five different types of frames exist:

1. Protocol header frame

2. Method frame

3. Content header frame

4. Body frame

5. Heartbeat frame

One important thing to note, is that the method frame (2), the content header frame (3) and the body frame (4) are all sent in order, upon publishing or consuming messages.

I will go through each one of them shortly, but let's first break down a frame, how it looks like and which parts make it.

## Frame internals

Every single frame is split into 3 parts

1. Frame header

2. Payload (frame body)

3. End marker

![](https://miro.medium.com/0*4PFo727OOqX-sqW4)

### Frame header

The header contains 3 parts

1. The frame type. It's just a single byte indicating the type of that frame, from the types I mentioned earlier.

2. The channel number. This is an integer number which uniquely identifies the channel. RabbitMQ can determine to which channel this frame is destined for.

3. Frame size. The total size of the frame in bytes.

![](https://miro.medium.com/0*eyC9_6YfkOwXvjwS)

### Payload

It encapsulates the body of the frame. For some frames, this part is encoded, for performance reasons, but for others it just raw data, either binary or string.

### End marker

It indicates the end of the frame, it's the ASCII value 206.

## The protocol header frame

This frame is only sent once and only aims to kickstart the communication between the client and server. This and the heartbeat frame should not worry developers, because the driver abstracts these away.

On the example shown in previous post, this happens on RabbitMqClientBase, line 32, when I create a connection with RabbitMQ.

{% gist 24731337ab03503ec4319e5bd564e74f %}

## The heartbeat frame

This is also a frame that's abstracted away by the client library. It is like a healthcheck, sent from and to RabbitMQ to ensure both sides are able to communicate. If the client application does not respond to the heartbeat request from RabbitMQ, the connection will be closed. The default interval is 600 seconds, which of course is configurable via the _rabbitmq.config_ file.

## The method frame

During publishing or consuming, the method frame is the first frame that's sent. It carries the command and required parameters related to that command. For example, the Basic.Publish command, which is about message publishing, would require the exhange name, routing key, mandatory and immediate flag parameters, which of course will be carried in the same method frame.

RabbitMQ will extract the command with its parameters from the frame and execute it. Let's see how a method frame looks like.

![](https://miro.medium.com/0*XMOb9-EeUGKc7qMV)

From the above, for that specific frame, we see that it starts with the class (Basic) and method (Publish) ids. Next, it contains parameter information, such as the exchange name and routing key, which will help RabbitMQ to determine to which queue to publish the message to. The mandatory flag says that the message must be delivered to a queue. If the message cannot be routed to a queue, either because the queue or the binding does not exist, then it returns a Basic.Return request back to client application. If you set that flag to true, you should handle the Basic.Return RPC send by the server in your code. The immediate flag tells the server how to react if the message cannot be routed to a queue consumer immediately. If this flag is set, the server will return an undeliverable message with a Basic.Return method. If this flag is false, the server will queue the message, but with no guarantee that it will ever be consumed.

In ProducerBase class, on the code example, on line 31, I'm issuing a Basic.Publish command to RabbitMQ which is going to send the 3 frames (starting with the method frame as showed above).

{% gist 0bb4a4d9186a33f0139c95d15dafbf12 %}

I've captured the RPC command that's sent using [tcpdump](https://hub.docker.com/r/kaazing/tcpdump) and then opened it using [Wireshark](https://www.wireshark.org/). In the image below we can see the method frame on the request packet.

![](https://miro.medium.com/0*cuThRRk26pUlu5Le)

## The content header frame

That's the next frame in order that's sent after the method frame. This one contains metadata information that describe the message and it's uber useful for other applications too, as these attributes can tell the code how to treat the message, leading to more intelligent consumers which define strict contracts for communication with other producers.

This frame contains information coming from the Basic.Properties table. In code, I would need to set the properties manually and add them to the RPC request, e.g. on [Channel.BasicPublish](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicPublish(System.String,System.String,RabbitMQ.Client.IBasicProperties,System.Byte%5B%5D)) method call which contains overloads that accept an IBasicProperties object. In the example on previous post, I did set few properties on Basic.Properties table, such as the app Id, content type, delivery mode and timestamp. Of course there are plenty of other properties available, check the table below which lists all properties. For more comprehensive list check the IBasicProperties [specification](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/type-RabbitMQ.Client.IBasicProperties.html). Please note, the reply-to-address is not listed in AMQP [specification](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/type-RabbitMQ.Client.IBasicProperties.html), it's a convenience property provided by the .NET driver.

app-id cluster-id (deprecated content-encoding content-type correlation-id delivery-mode expiration headers message-id persistent priority reply-to timestamp reply-to-address* type user-id

The producer's code, sets the properties using the CreateBasicProperties method of channel object.

{% gist b85f1b8d0cd66491e9b80f756203d160 %}

Let's break down the previous content header frame sent by the producer.

![](https://miro.medium.com/0*_4xIAUnMYgUHO8-a)

One thing to note here is the body size, which is the first property of the content header frame. If that exceeds the max frame size defined by RabbitMQ, then the contents of the message will split in several body frames and each will be sent in order. The next property defined is the property flags that tells RabbitMQ which properties have been set.

The rest of the properties set are pretty self-explanatory and it all depends on which properties are set by the application code. Following is the captured package for the Basic.Publish command, showing the content header frame with its message properties defined.

![](https://miro.medium.com/0*CB8nGXGvx6BB7LH4)

## The body frame

That's the final frame sent during producing/consuming and the most important one, as it holds all the beef, i.e. the actual content of the message, which might be binary or text.

![](https://miro.medium.com/0*QBLWaJXs6L3im900)

In similar fashion with the above, Wireshark can help me inspect the body frame of the same Basic.Publish command. The payload is serialized into JSON format.

![](https://miro.medium.com/0*vIR1BQju8huu7NS7)

## Communication while publishing and consuming messages

To publish or consume messages to/from RabbitMQ, an application needs to

1. Declare an exchange

2. Declare a queue

3. Bind the queue to the exchange

Let's see how this communication unfolds. In RabbitMqClientBase class, which is common for both my producer and consumer apps, I'm creating an exchange with the following code.

{% gist ef9f3c22171a419a014f3d80a65359d6 %}

Doing this, the client sents an Exchange.Declare command to the server, with the latter replying with an Exchange.DeclareOk response. The command tells RabbitMQ to create an exchange, with the name provided in the arguments. If that exchange already exists, RabbitMQ will do nothing.

![](https://miro.medium.com/0*OSPllMBv8S7RMPoO)

Following is the captured traffic in Wireshark.

![](https://miro.medium.com/0*66ujMhy7BvYmmjsi)

Next, the queue must be created. Similarly, a Queue.Declare command is sent from the client, telling RabbitMQ to create a queue. This request is idempotent as well, meaning, if the queue already exists, no action will be taken. RabbitMQ responds with Queue.DeclareOk.

{% gist cb1a0fb6963d33807848bb1946cb5b64 %}

The code above initiates the following conversation.

![](https://miro.medium.com/0*9lHmuDaGZlEDzZmd)

And here's the captured traffic in Wireshark, confirming the above.

![](https://miro.medium.com/0*4GwoJK56j2MYF4g3)

Finally, the queue should bind to the exchange. The same communication pattern occurs here as well, a Queue.Bind command is sent by the client, and the server replies with a Queue.BindOk response. This is idempotent too.

{% gist c1c64cf0091297e5042ea05f99fae5f3 %}

Calling the QueueBind method on the .NET client, produces the following conversation.

![](https://miro.medium.com/0*AT3thdJrxTa9JlOD)

And here's what I've captured in Wireshark.

![](https://miro.medium.com/0*poVLC8Quz79MBvrF)

### Preparing the consumer

To start consuming, the .NET client must handle the Received event of the consumer object and call the [BasicConsume](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/type-RabbitMQ.Client.IModel.html#method-M:RabbitMQ.Client.IModel.BasicConsume(System.String,System.Boolean,RabbitMQ.Client.IBasicConsumer)) method on the channel.

{% gist a9d7af8a1311794ca3cc72fb1d71b655 %}

The latter sends a Basic.Consume command to the server, telling RabbitMQ to deliver any incoming messages from that specific queue, defined in the arguments, back to that consumer. RabbitMQ responds with a Basic.ConsumeOk after it executes the Consume method.

![](https://miro.medium.com/0*pTWqc0lN8_8n06vt)

And the capture on Wireshark

![](https://miro.medium.com/0*C1g-0AeT72FSSuxO)

New messages are published to a queue with the Basic.Publish command. When the server receives that, it inspects the method frame to extract the exchange name and routing key. With that information in hand, it can match the exchange and evaluate the bindings in the exchange. With the bindings been evaluated, the broker knows to which queues this exchange is attached to, so the only thing that is left is to match the routing key for each queue with the routing key provided in the message properies. If the queue is matched, then the message is dequeued from that queue and sent to the consumer(s), with the broker issuing a Basic.Deliver command to interested parties.

The consumer will be receiving messages until it is disconnected or until a Basic.Cancel command is sent to the broker. In the code snippet above, I've set the autoAck property to false, which means the consumer must send a Basic.Ack command to the broker.

{% gist f45687654b72d32455ce5689539cc350 %}

It is required to acknowledge the message, so the broker knows if it should be dequeued or not. If the autoAck property was true, it wouldn't be needed to return an acknowledgment, the message would be acknowledged automatically the moment it was consumed. This of course depends on your application needs, if you need more control, set the property to false and manage acknowledgements in the code.

Something also worth noting here, is the delivery-tag, which is passed as an argument with the Basic.Ack command. I've picked this delivery tag from the Basic.Deliver request and I've provided it to the Ack method. This is required, because RabbitMQ will use the channel id along with the delivery tag to handle acknowledgments.

![](https://miro.medium.com/0*spatEATuMOYNrp89)

### Publishing a message

The producer is now ready to publish some messages, same goes for the consumer, which is up and running and is listening for messages on the queue. Let's see how it looks like, on the wire, when a Basic.Publish command is sent and how the 3 components of the demo system (producer, server and consumer) interact with each other.

![](https://miro.medium.com/0*6yw4QNka9OFgSL7n)

From the above, we see that the server, as soon as it enqueues a freshly published message to its queue, it looks to deliver it to the corresponding consumer, making a Basic.Deliver RPC request to the client. The following capture in Wireshark proves that point.

![](https://miro.medium.com/0*25yS4sxd3RTgnbTX)

The Basic.Publish command originates from 172.20.0.4, which is the producer application. Its destination is the RabbitMQ server, which address is 172.20.0.2.

Next, the server from the same 172.20.0.2 address, sends a Basic.Deliver command to the consumer application, on address 172.20.0.3.

Finally, the consumer client application sends a Basic.Ack back to the server.

## Summary

The book [RabbitMQ In Depth](https://www.amazon.co.uk/RabbitMQ-Depth-Gavin-M-Roy/dp/1617291005) has been a great inspiration for this post, it's an excellent guide, I would say the most comprehensive source for RabbitMQ out there. I definitely recommend it if you are interested in learning or currently using RabbitMQ for enterprise level applications.

For a detailed guide on the RabbitMQ .NET client, please check the [API documentation](https://www.rabbitmq.com/releases/rabbitmq-dotnet-client/v3.6.14/rabbitmq-dotnet-client-3.6.14-client-htmldoc/html/index.html).

Full code of the sample producer and consumer applications can be found on my GitHub [repository](https://github.com/gdyrrahitis/bpost-rabbitmq).

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).
