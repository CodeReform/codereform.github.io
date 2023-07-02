---
layout: post
title: Configuring an Nginx TCP proxy for my RabbitMQ cluster in under 10 minutes
date: 2020-10-17 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- "RaspberryPi"
tags:
- RabbitMQ
- AMQP
- RaspberryPi
- Nginx
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2020/10/17/configuring-an-nginx-tcp-proxy-for-my-rabbitmq-cluster-in-under-10-minutes"
---
![](https://miro.medium.com/1*EP7jWZsvNoRYw2KcyKdb2g.jpeg)

# Configuring an Nginx TCP proxy for my RabbitMQ cluster in under 10 minutes

In the [previous post](https://medium.com/swlh/how-to-build-a-rabbitmq-cluster-with-a-few-tiny-raspberry-pi-zeros-e5ffb3920e40), I described what a cluster is and how to setup a RabbitMQ cluster on Raspberry Pi with a few Zero W's. I've built a 5 node cluster, with one master and 4 followers. But this is not enough, I'd like my cluster to be accessible from a single location, it doesn't really matter which node my applications connect to, since I would work only with HA or Quorum queues. For the latter, a new blog post is coming up where I'll go through exhaustive details on these queues, their pros, cons and usage examples.

## Introduction

In this blog post, I will add a proxy in the mix by adding a new Raspberry Pi Zero W to the existing setup. I will install Nginx on that device and configure a TCP proxy.

## Requirements

I will need an extra Raspberry Pi Zero W. Thankfully, I already had a spare one, used mostly for prototyping projects.

## Nginx

Nginx is an open source web server that can also be used as a reverse proxy, load balancer, HTTP cache and mail proxy. It is very popular among web developers, with thousands of active websites across the internet currently being served by Nginx.

Nginx is an extremely powerful tool, which is not only popular as a webserver but also as a proxy and this is the reason I picked it for this job. What I want is to proxy incoming connections to any of the 5 RabbitMQ nodes on my tiny cluster. I know that Nginx doesn't have a problem to proxy HTTP requests but what about TCP, which RabbitMQ uses to communicate? There is no problem on that either, as Nginx includes the [stream module](http://nginx.org/en/docs/stream/ngx_stream_core_module.html), which can be used to load balance TCP or UDP connections.

## Installing Nginx on Pi

I wish to only use the stream module for now, so the prebuild debian package would do just fine for me. I also don't need bleeding edge features, so the stable version would do the trick. If I needed any new features or experimental modules I could use the mainline version but I'm not interested for now.

You might ask, what is that prebuild version? Well, that's the easiest way to install Nginx with all the default modules enabled. If I needed to add any other 3rd party module for example, I would have gone on a different route, which is more complex and would involve compiling from source. I won't discuss that here as it is out of topic.

So, to install Nginx I had to run the following 3 commands in order.

First update the package lists

{% gist 0485e4f7dcb1a986859516dbf7d8fff7 %}

Then install Nginx

{% gist 9662b7a2be30378551868972e6aae92e %}

Test if Nginx is successfully installed

{% gist 59060744536d7532549cae076eba7305 %}

## Configuring TCP proxy

The nginx.conf file needs to be updated in order to set a new TCP proxy. The file is located at

```
/etc/nginx/nginx.conf
```

Before I start editing, let's do a quick recap on the upstream servers. The cluster is made of 5 nodes, all accessible via the following IPs in the WLAN network.

![Nodes in cluster](https://miro.medium.com/1*cKvPS1JyIzmQirfB4i5Law.png)

By looking at the listening ports on the management portal, the ones that I'm interested in using, are, the AMQP protocol **_5672_** port and the STOMP protocol **_61613_** port. The http/web-stomp is configured on port 15674 and uses websockets, but I am not interested into that one for now. In a future blog post I will show you how I did set this up as well. Finally, I'd need to proxy HTTP requests to the 1**_5672_** port for the management portal.

![Ports and contexts](https://miro.medium.com/0*AStDCMqrsVxpc-Mr)

Now it's time to edit the nginx.conf file. To configure a TCP proxy, use the **_stream_** top level block. Inside that, one can define one or more **server**blocks. In these blocks we usually define the port to listen from and an up**_stream_** group of servers to proxy the connection to. It is also the best place to define [proxy configuration.](https://nginx.org/en/docs/**_stream_**/ngx_**_stream_**_proxy_module.html?&_ga=2.165675705.620468574.1602789080-1123877840.1602789080)

The **_upstream_** group of servers is defined using the **_upstream_** block inside the stream top level block. One can list the servers to proxy the TCP connection to, but also the load balancing method to use. Let's see how all these are defined below.

{% gist 6d6bf76a4b20ef49f824089fac190a3c %}

In the snippet above, I've configured two **_server_** blocks and two **upstream****_server_** groups. One **_server_** block defines an AMQP while the other a STOMP virtual **_server_**. The first listens to 5672 and the latter on the 61613 ports on the Nginx **_server_**. The **_proxy_pass_** directive has the alias of the **upstream**group of **_server_**s. Both are defined above with their respective names.

In the upstream groups, I've first defined the load balancing method. Nginx provides several of them but to be honest, I am not particularly interested which **_server_** the proxy would choose. The reason is, I will mostly use HA or quorum queues, which allow the cluster to create replicas to other nodes, so there is no problem like cross-node communication, all nodes have a replica of the same queue. I was between random and[ least connections](https://nginx.org/en/docs/stream/ngx_stream_upstream_module.html?&_ga=2.122078988.620468574.1602789080-1123877840.1602789080#least_conn) but I choose the latter for obvious reasons. The[ least connections](https://nginx.org/en/docs/stream/ngx_stream_upstream_module.html?&_ga=2.122078988.620468574.1602789080-1123877840.1602789080#least_conn) load balancing algorithm will select the **_server_** with the smaller number of current active connections, which is at least fair for the nodes in the cluster. Finally, I've listed all nodes with the **_server_** directive.

Almost ready, before we jump into testing we got to first reload the Nginx configuration to take place and this is done with the following command.

{% gist ea1843529d311e93e333466699d2ed53 %}

### Check error or access logs if something goes south

If something goes wrong go to the following location and check the access.log and error.log files.

```
/var/log/nginx
```

The access.log file contains information on incoming HTTP requests, while the error.log records any problem occurs with Nginx, for instance bad configuration or inability to connect to an upstream server.

## Configuring an HTTP reverse proxy for management portal

A reverse proxy is essentially an entity that sits in front of one or more web servers. It provides a seamless communication between clients and the servers that hide behind it.

To configure the reverse proxy, it's similar process as the above, but on this one I have to create a rule inside the **_http_** top-level block. This one handles HTTP traffic, compared to the stream block which handles TCP traffic as we saw earlier.

{% gist 5b1bd9d9fc82cd09248a20ebc8fe9593 %}

On the above, I've defined a server block which listens to port 15672 on the default server. A request on the root path (/) is proxied to the management portal on the master node.

## Quick test with a small publisher

I have created a publisher to test the above. It connects to the proxy server and prints whether the message is delivered or not. To let the publisher know if the message is delivered, I have to enable **_publisher confirms_**.

### Publisher confirms

This is a lightweight alternative to transactions and provides a certain level of message delivery guarantee for interested consumers. Please be advised that not all client libraries support publisher confirms, but thankfully, Python rabbitpy does. It works by sending a Basic.Select RPC request to RabbitMQ to enable publisher confirmations. The server responds with a Basic.SelectOk. After that, for every Basic.Publish that is issued, the server responds with a Basic.Ack, if successful, or a Basic.Nack otherwise.

Please keep in mind that publisher confirms can have a negative effect on the publisher's performance, as it will block until the confirmation is received. Also, don't use publisher confirms with transactions, as both don't work together and you will instead get an error.

### Mandatory

I've also set the **_mandatory_** flag to True on publish, in order to make sure that my message will be routed. If I make a mistake on the routing key for example, my message will never reach its destination, but with **_mandatory_** flag set, if the message is unrouted, an exception will be thrown.

This is the publisher to test whether the TCP proxy works:

{% gist c763fae2978370f05f061c8089eb1db6 %}

When I run the publisher, I get the message shown below.

![](https://miro.medium.com/1*DsnWvUWVU6xbJrZ8IA-Hew.png)

Seems that everything is working! Same goes for the management portal (I created an alias on hosts file on the 192.168.0.221 IP)

![](https://miro.medium.com/0*pYKQ9LDRFwCcuDHK)

## Summary

In this post I described what is Nginx in a nutshell and configured a TCP proxy on my Raspberry Pi Zero W. Nginx is an extremely powerful webserver which is very easy to install and use. Finally, I tested my setup with a small RabbitMQ publisher and I described what publisher confirms is and how it guarantees message delivery, along with the mandatory flag on publish.

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).
