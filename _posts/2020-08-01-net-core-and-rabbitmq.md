---
layout: post
title: .NET Core and RabbitMQ
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
permalink: "/2020/08/01/net-core-and-rabbitmq"
---
![](https://miro.medium.com/1*QwkJeq13EEaXk7p1nOLibg.jpeg)

# .NET Core and RabbitMQ

Having a system which is composed by distributed applications is a great idea, but a way to communicate with each other is required. A very popular architecture is the so called MDA or Message Driven Architecture, where a system is composed from autonomous components that communicate with each other via messages. The part which facilitates communication is the message broker, effectively decoupling applications, which don't communicate directly, rather they publish messages to the message broker and the latter is responsible to forward them to interested parties, i.e. other applications.

A message broker that is particularly powerful and interesting is RabbitMQ, one of the most popular open source tools for that job, used worldwide by large enterprises to small startups.

In this post, I'm going to explore RabbitMQ's basics, by creating a simple RabbitMQ producer and consumer in .NET Core with C#. Then, I will take discussion to how AMQP works at a low-level, using the code example demonstrated.

## Creating a producer and consumer application in .NET Core

I am going to create a producer application, which publishes messages to the broker and also a consumer application, which consumes messages from a RabbitMQ queue.

One fine way to create a producer and a consumer application in .NET Core is to use [Hosted Services](https://docs.microsoft.com/en-us/aspnet/core/fundamentals/host/hosted-services?view=aspnetcore-3.1&tabs=visual-studio). A hosted service is a background task, which comes in three flavors:

1. Scheduled

2. Long running task

3. Queued

We're not going to be bothered with #1 and #3 in this post. I am going to create a hosted service for the producer application first. This hosted service is going to connect to RabbitMQ, declare an exchange, a queue and bind them. Every 20 seconds, the producer is going to publish a message to the queue.

Regarding the consumer, again, I am going to create a hosted service, which also is going to connect to RabbitMQ, declare an exchange (the same exchange as previously), a queue (the same queue) and bind them (same binding). As this is a simple consumer application which just logs the consumed messages.

## Connecting to RabbitMQ

First thing first, install the [RabbitMQ.Client](https://www.nuget.org/packages/RabbitMQ.Client) nuget package, this is an implementation of the AMQP 0–9–1 client library for C#.

In order to connect to RabbitMQ, the client library provides a ConnectionFactory class, which API contains a method called CreateConnection. This allows the client to connect to RabbitMQ and also perform the rest of the required steps which are

1. Open a channel between the client and the broker

2. Declare an exchange

3. Declare a queue

4. Bind the queue to the exchange

To create a connection factory it's pretty simple, you just need the RabbitMQ connection string and voila! The connection string below follows this format:

amqp://{username}:{password}@{host}:{port}/{virtual_host}

I've omitted rest of registrations as they are not important on this demonstration. Be advised, I'm going to create two different client applications, so I've created a connection factory on both of them. As a note, the producer configuration is slightly different, due to the fact that my producers are asynchronous. More on that later.

{% gist f9c4584ae1c7a6123e3e6582bd3823d1 %}

Now back to the really important bit. I've created an abstract base class for connecting to RabbitMQ and then added that to a common class library. Reason is I want to reuse the same logic across my producer and consumer applications. All consumers and producers are going to derive from that base class.

{% gist 75cfe1b97eb7b5bb0b290d1d6a698522 %}

On breaking down the above, on ConnectToRabbitMq method, I'm first looking if that connection is already established, if yes then no reason to open this again.

Next, I'm doing the same for Channel. To open a channel, I'm using the CreateModel method on the connection object I've created earlier from the factory.

Finally, I declare an exchange with the ExchangeDeclare method with some arguments, which are briefly discussed below.

- The exchange name.

- The type of the exchange. I've chosen direct, but there are other types as well, like Fanout, Topic and Headers

- Durability of the exchange. Means that the exchange will not be deleted if RabbitMQ restarts.

- Auto delete instruction, which makes the exchange automatically delete if all of its queues unbind.

I've also declared a queue, with the following arguments.

- The queue name.

- Durability of the queue. This is whether the queue will survive when RabbitMQ restarts.

- Exclusivity of the queue. This tells the queue to be used by only one connection and automatically delete when that connection closes.

- Auto delete instruction. Means that the queue, given that it had at least one consumer, will be deleted when all of its consumers unsubcribe.

And then I've bidden the queue to the exchange with the following arguments.

- The queue name

- The exchange name

- The routing key. Incoming messages will have their own routing keys, the exchange will use the routing key on its binding to evaluate incoming messages and deliver to the appropriate queue.

## The producer

For the producer application I have created an empty ASP.NET Core project, with a class that that implements BackgroundService. This is a long running hosted service and it's responsibility is to publish a message to the broker queue every 20 seconds. As long as the task is not cancelled it's going to run forever. Let's look at that first.

{% gist 48b83bb14b6ef3220d4afc9706dc5706 %}

The LogIntegrationEvent is the message carried from one application to the other with RabbitMq acting as a medium between them. As you can see below it's just a class with couple of properties and nothing special about it.

{% gist caa876770ef63be157906b3377d10871 %}

What is particularly interesting though is the producer. Let's see first producer's contract which just exposes a Publish method. Each producer publishes its own type of message, so it's essential to make the publish method generic in order to publish a variety of messages.

{% gist 2a3eb9efbf388cdc6af65f73530fbf2e %}

Next, I've created an abstract base producer class, which contains all required logic to publish a message. I can implement different producers by deriving from this class. Driver's BasicPublish method issues an RPC command to publish a message on that channel. RabbitMQ requires to know to which exchange you publish that message and what is the routing key. By having these two, the broker can determine where to deliver that message.

{% gist cf48b58bd8fa06dc0ddbf373823ec526 %}

Finally, I've created a LogProducer, which derives from the ProducerBase class. I've also provided the exchange name and routing key required for that particular producer. Using that base class I can create any producer, as long as I can provide information on the exchange, the routing key and the generic parameter, which is the integration event, a.k.a. the message.

{% gist e9b737316af210906e6afae8b9cd8c15 %}

## The consumer

To consume the messages that are delivered by the publisher above, I need a consumer application to consume from an existing queue. I've created an ASP.NET application with a HostedService, in which I perform the actions described above. The following is the log queue specific consumer.

{% gist ecfb4c4ffb99a8d339ff36760a2c9afe %}

This is a simple background task and to make one I just need to implement the IHostedService interface.

This consumer handles asynchronous code, so I had to create one using the AsyncEventingBasicConsumer class, passing the current open channel as an argument. To make the channel consume messages from a specific queue, I need to call the BasicConsume method and provide the queue name and the consumer object as arguments. The autoAck argument tells the channel whether to automatically return an acknowledgment back to the broker. This is important, as it tells the broker that the message is indeed consumed and doesn't need to be in the queue anymore, effectively dequeued. In this scenario, I've decided to sent an ack manually back to RabbitMQ.

Please note, it is required to set the DispatchConsumersAsync property to true on connection factory initialization, if any of the consumers is of type AsyncEventingBasicConsumer.

{% gist fb6765cc28e182a1bcf5a9d50f6b7940 %}

Finally, I need a way to handle the incoming messages, so for this I'm handling the Received event on the consumer. This event is invoked only when new messages enqueue on the target queue. The implementation lies on the following base class.

{% gist 2b60a3007490283b6e286db7abf2e16c %}

I've serialized the body, which comes in a binary format, to the appropriate command, which is just a plain C# class. Notice also that I'm using the [MediatR](https://github.com/jbogard/MediatR) library, which implements the mediator pattern beautifully and it's particularly useful here. I've created a command handler for the LogCommand, which runs some business code when the command is send to the mediator. This instantly decouples the consumers and the business logic required for each incoming message.

To create a command, a plain C# class is needed, with few properties of course to store the data. Plus it's required to implement the IRequest interface from MediatR library.

{% gist a1e51ebfdf285bb3f39f52c7af859501 %}

Finally, I've created a command handler which implements the IRequestHandler interface. Notice, that the generic parameter is the command class I've created earlier, this is to tell MediatR which command this handler is going to handle. In the Handle method, I'm logging the incoming message from RabbitMQ.

{% gist 71193de1d8e6dc400771ae751eedd625 %}

## Running the applications with docker-compose

The code is ready, it's time to start the applications and consume the messages. I love docker because it makes things so easy for a developer, with one command I'm spinning up both consumer and producer applications and I also start a RabbitMQ server. Imagine the pain if no docker ever existed, I would have to run each application separately and install RabbitMQ on my machine. Ugly.

Following is the docker-compose file to run my applications, which only contains definitions for the images. The override file contains specific definitions for each container. This is very useful if you want to have different definitions for local, development and production environments for example.

{% gist faae775ac400612869a8c9f13023561b %}

Following is the override file, which is pretty standard. However one thing that stands out is the depends_on definition on each ASP.NET application. This tells the containers to not start until RabbitMQ container enters a healthy state.

> _I'm using v2.4 on docker-compose as in 3.x depends_on is deprecated._

{% gist ad86ba94839bd277a772311cc2967350 %}

To run the apps, I'm running the following command on the root folder.

{% gist 666f0e0595633918171966e956841cdf %}

It works pretty well, I can see messages being consumed (open full screen to see the logs).

<iframe src="https://cdn.embedly.com/widgets/media.html?src=https%3A%2F%2Fwww.youtube.com%2Fembed%2Fmahxh9GmQOY%3Ffeature%3Doembed&display_name=YouTube&url=https%3A%2F%2Fwww.youtube.com%2Fwatch%3Fv%3Dmahxh9GmQOY&image=https%3A%2F%2Fi.ytimg.com%2Fvi%2Fmahxh9GmQOY%2Fhqdefault.jpg&key=a19fcc184b9711e1b4764040d3dc5c07&type=text%2Fhtml&schema=youtube" title="" height="480" width="854"></iframe>

## Why?

Why all these steps? How it works behind the scenes? I am interested to know how RabbitMQ communicates with the AMQP protocol on a low-level.

Let's look at AMQP and how this dance between the broker and client unfolds on the [next post](https://medium.com/swlh/net-core-and-rabbitmq-part-2-communication-via-amqp-35c5518cb64).

Full code of the above example can be found on my GitHub [repository](https://github.com/gdyrrahitis/bpost-rabbitmq).

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).
