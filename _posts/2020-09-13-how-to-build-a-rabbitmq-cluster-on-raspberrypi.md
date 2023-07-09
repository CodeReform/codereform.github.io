---
layout: post
image: https://miro.medium.com/1*HCqgUn5K3qSSARwIBniHTg.jpeg
title: How to Build a RabbitMQ Cluster on Raspberry Pi
date: 2020-09-13 00:00:00.000000000 +01:00
type: post
parent_id: '0'
published: true
password: ''
status: publish
categories:
- "RaspberryPi"
tags:
- RabbitMQ
- RaspberryPi
- AMQP
author:
  display_name: George Dyrrachitis
  first_name: George
  last_name: Dyrrachitis
permalink: "/2020/09/13/how-to-build-a-rabbitmq-cluster-on-raspberrypi"
---
![](https://miro.medium.com/1*HCqgUn5K3qSSARwIBniHTg.jpeg)

# How to Build a RabbitMQ Cluster on Raspberry Pi

In this blog post I will show you how to build and configure a 5-node Raspberry Pi cluster and use RabbitMQ's clustering capabilities on the above to scale the message broker horizontally.

## Introduction

Experimenting with Pi clusters is something that I have been thinking a lot lately. I decided to build a small Pi Zero W cluster for fun and experimentation, in order to fiddle around with messaging mechanisms such as [MPI](https://en.wikipedia.org/wiki/Message_Passing_Interface) or to manage Kubernetes or RabbitMQ clusters. In this post, I'm going to talk about the latter.

## Requirements

You need to purchase some hardware if you intent to fully follow the tutorial, otherwise you can always create VMs using Vagrant or other tools. I will stick with Raspberry Pi as it is fun and because I've been experimenting a lot lately with these little fellas!

Here's a detailed list of hardware requirements for a 5-node cluster (you can always purchase more or less Pi's if you prefer a cluster of a different size)

- Raspberry Pi Zero W.

- [A USB power supply](https://www.amazon.co.uk/gp/product/B01K7F51VU/ref=ppx_yo_dt_b_asin_title_o07_s00?ie=UTF8&psc=1) (USB output should provide at least 2.0A, the Pi Zero W requires around 1.4A).

- 5 micro USB cables.

- 5 Heat sinks. You don't want your Pi's to overheat!

- Raspberry Pi Zero [Cluster case](https://thepihut.com/products/mini-cluster-case-for-raspberry-pi-zero). Important if you want your cluster to be tidy and clean.

Total cost of the above is 155€. You can always purchase a cheaper USB power supply, the one I'm suggesting above cost me 61€, but I purchased it mostly because it looks slick and goes well with my desk setup. Only remember to purchase something that gives at least 2.0A on each output. And as a final note, this USB power supply won't work well with Raspberry Pi 4 or 3 models, as they have different power requirements, please check the [official page](https://www.raspberrypi.org/documentation/hardware/raspberrypi/power/README.md) for more details. In such clusters it's more advisable to use an [ATX power supply](https://en.wikipedia.org/wiki/Power_supply_unit_(computer)) or something similar. However, for the purposes of this Pi Zero cluster, the power supply above is more than enough.

## A bit of theory…

RabbitMQ is great for most applications, but what happens in enterprise? Such applications would require additional delivery guarantees and wouldn't be able to handle transient failures or server outages. Just imagine a mission critical system that goes down often due to datacenter outages, network issues or other factors, acting as a single point of failure. The amount of dollars that'd be lost in each incident could be catastrophic for any reputable organization. Some C-level executives might have a stroke upon hearing the losses. Well, and developers would be most likely fired :|

Clustering in RabbitMQ can be particularly useful as you might have noticed. The problems listed above could not pose a threat anymore and we all can live happily ever after. Jokes aside though, let's discuss what is exactly a cluster.

A computer cluster is made of many tightly coupled computers that usually share the same local network and work together to tackle computing tasks. If one views the cluster from the outside though, won't be able to tell if multiple computers are working together, it mostly resembles one single system from that point of view. Computer clusters have many advantages over single computers and this point is further enhanced when we talk about enterprise applications that serve millions of users daily. Mainly, clusters have the following advantages

- Performance improvement for compute-intensive systems

- High availability

- Fault tolerance

- Scalability

- Inexpensive compared to other paradigms

However, the above comes with the cost of extra complexity. The bigger the cluster, the more complex it becomes and harder to maintain, except of course if there is a proper plan or automated system that takes care of maintaining and patching the cluster nodes.

## Clustering in RabbitMQ

Given the advantages listed above, we can deduce that computer clusters would be particularly useful for a message broker like RabbitMQ. Think about it. Let's say that I have a system that has strict requirements in message delivery, no message loss is tolerable. Now, is a single RabbitMQ instance enough to guarantee the above? I believe no, because there are always risks, a datacenter, where the computer that has RabbitMQ installed, might go down, or the particular server or VM can go down, many things can go wrong, we shouldn't trust the infrastructure to work properly, given if the system has the above requirements of course. The single instance becomes a single point of failure, a bottleneck for the bespoke system. Although RabbitMQ offers several delivery guarantees, they are simply no good if the server cannot accept/deliver messages.

The above issues however can be easily addressed with a cluster. The single point of failure is now gone, with multiple nodes able to handle consumers and publishers and if any of them goes down, for example due to maintenance, the other nodes of the cluster can still pick up the work. One or two dead nodes cannot stop the system's continuity, it can still operate like there were never down! Another advantage of RabbitMQ clustering is that applications can have access to additional delivery guarantees, like HA queues.

In the RabbitMQ cluster, the runtime state, often called shared state, contains the following, which are shared and available to all nodes

- Exchanges

- Queues

- Bindings

- Users

- Virtual Hosts

- Policies

There are several ways one can create a RabbitMQ cluster. One is to use a cloud provider like AWS or Azure and setup a cluster there. Another way is to setup a cluster on-premise, however one has to keep in mind that the environment the cluster should be set up should not be across WAN or the internet, but only within a low latency WLAN network. For high-latency environments, additional tools should assist in configuring the cluster.

## Cluster node types

In RabbitMQ you will find two node types

- The _Disk node_, where runtime state of the cluster is stored to both RAM and _Disk node_s.

- The _RAM node_, which stores the runtime state in memory.

Clusters should have at least one node defined as a Disk node in order to be able to restore the previous (prior to failure) state. Having too many disk nodes however can be problematic as well. Can you guess why? Yes, disk I/O may contribute to performance issues. However this is not the only issue! If many disk nodes go down, it might be not easy for them to automatically bring themselves back, as they might not agree on the runtime state, if it has changed while they were down. When the disk nodes come back, they will recreate their previous state and sync with the other nodes in the cluster. If the runtime state has changed, they will not be able to bring themselves back, so manual intervention might be needed. RAM nodes on the other hand don't experience any of these issues, because when they are brought back in the cluster they recreate their state from scratch by syncing with other nodes.

## Erlang and erlang-cookies

RabbitMQ uses Erlang's multi-node communication under the hood. To achieve this, it has to use a secret shared file called cookie. That file contains a string value which must be the same across all nodes in order to communicate with each other. This file is created the first time RabbitMQ server starts or if the file is missing.

You can find that file at /var/lib/rabbitmq/.erlang.cookie. I am going to set a specific cookie value later for all nodes in the cluster.

## Cluster setup

I will not show you how to assemble the cluster case, as you might as well have purchased a different one or you might haven't purchased at all and you're reading this only out of curiosity. These things come with a manual and sufficient instructions so you should be okay.

Regarding my Raspberry Pi's, I've placed a heatsink on each one of them. For those who don't know what a heatsink is, it's just a passive heat exchanger that transfers heat from an electronic to another medium, such as air. This helps the electronic device to retain a normal level of temperature. I used the heatsink to avoid my Pi's overheating while running.

![5-node Raspberry Pi Zero W cluster](https://miro.medium.com/0*_54HNhFUFw93zkQB)

A Raspberry Pi doesn't come with a hard drive or anything like that, it requires an SD card, in which one can install the OS. I've installed the [RaspberryPi OS Lite](https://downloads.raspberrypi.org/raspios_lite_armhf_latest), as I won't be needing any desktop features. In addition, the desktop image would put my Pi's under stress, as the runtime and libraries included in such images tend to consume more resources and I would like to save as much as I can as the Pi Zero W has limited RAM and processing power.

To install the OS, I personally use my other Raspberry Pi 4, I plug in the SD card and follow the installation steps. I just find that quick and easy to perform.

## How to reinstall RaspberryPi OS

In case you have different image installed and would like to reinstall the RaspberryPi OS, in order to install Lite that is, you can follow the steps below, else jump to the next section.

First, connect to your Raspberry Pi, using [PuTTY](https://www.putty.org/) in Windows or SSH on [Linux](https://phoenixnap.com/kb/ssh-to-connect-to-remote-server-linux-or-windows). Then download the [RaspberryPi OS](https://www.raspberrypi.org/downloads/raspberry-pi-os/) Lite image using the following

{% gist 6829718a8f9db4f11d27f8748f7a0ea5 %}

Next, unzip the file. In the command below, I'm unziping the contents into a new directory named "image".

{% gist 5a17bef42fe110a1b78656a90e790db9 %}

Then, it's important to find the current mountpoint and partitions on the SD card. I have to unmount any mounted partition I find, however I don't have any, so moving forward.

{% gist e77f03cec2d4041e13f5900a769613da %}

The above will give use the device name and partitions. I will use the device name to copy the image to the SD card. The device name in my case is /dev/mmcblk0.

![](https://miro.medium.com/0*WjLgCLoouVHtGVzm)

And finally, it's time to copy the .img file, which is going to be installed next time I boot my device.

{% gist 3f0a9b03047f83bc38837ee40ea34e5e %}

After the above is done, the Pi must be rebooted in order to install the image.

{% gist 13922e91a423f8989e9da58d703525c4 %}

Device booted and back in action! However, the Pi Zero W doesn't have any ethernet ports, I guess I have to connect to the WiFi manually, but thankfully it's easy to do so by running the following command that opens the Pi configuration panel.

{% gist 7e633fa01f0b594d43472a9cd4942b6e %}

From there, all I have to do is to open _Network Options_, then _Wireless LAN_ and add the SSID and password.

![](https://miro.medium.com/0*DQVaJXRxksMSY576)

![](https://miro.medium.com/0*LC5dJhAzdKtTX3EV)

Congrats, the Pi is now connected to the internet! Moving to the next section now, it's time to install Erlang and RabbitMQ! Detailed steps to install OS on Linux can be found [here](https://www.raspberrypi.org/documentation/installation/installing-images/linux.md).

## Inter-node communication without password

An equally important step, prior moving into RabbitMQ installation, is to take care of the inter-node communication in the cluster. What I want to achieve here is to make each node accessible to all other nodes in the cluster. By doing that, I will be able to join nodes in the RabbitMQ cluster and also support inter-node communication across them.

Taking a step back I have to consider what kind of hardware I have in hands. At the moment, my cluster consists of 5 Raspberry Pi Zero W devices, which makes it not possible to setup an ethernet local network, rather all nodes must have their IP addresses assigned to a local DHCP server and communicate via WiFi. Remember, Pi Zero W comes with WiFi support out of the box. Fortunately, this task was rather easy, I used my wireless router's DHCP server and assigned a local IP address to each Pi. To distinguish each device I made sure to name each one of them uniquely using the sudo raspi-config command, by opening the _Network Options_ and _Hostname_ configuration.

![](https://miro.medium.com/0*vSj9d2fJeYUKXe3w)

![](https://miro.medium.com/0*Fl3VktISR_Ksn1yN)

In Linux, remote communication is available through _SSH_, so I also have to enable that in the config as well. To do that, open _Interfacing Options_ and _SSH_.

![](https://miro.medium.com/0*SIgARiqHbG7KpSfQ)

![](https://miro.medium.com/0*ZQydIhvlF04RzzVO)

With the above enabled, I can start manually making each device visible to the others. Let's recap on the cluster details. The cluster consists of nodes, each assigned a unique IP on DHCP.

![](https://miro.medium.com/1*LKOdzde51KWTFuuTCDYIrg.png)

## Setup SSH

The example below is only between rpi-node1 and master nodes, however the same process must be repeated for all node permutations.

I've connected to rpi-node1 via PuTTY from my PC. I have to create a new SSH key, copy it to all other nodes (in this example I will copy only to master node) and then append the key to authorized_keys file in target nodes.

I execute the following command in rpi-node1 node.

{% gist 15cb8a88e4086f1260c71a38c4d63dce %}

With the key generated, I copied it to the rpi-master node with the following command. I was asked to provide a password to connect initially but it won't be needed after I append the key to _authorized_keys_ file on master.

{% gist 6156899b7029d6bd08d226e1dd28c426 %}

I only now have to remote into rpi-master node to finalize the setup. In rpi-master node then I execute the following.

{% gist 308ca9e14504916014edb1dc0ee60b62 %}

Mission accomplished! The rpi-node1 node can now connect to rpi-master without providing password. To confirm this, I can demonstrate an SSH session to rpi-master from rpi-node1 and see if password will be prompted.

{% gist 8cbb21f022da8a2c58f05f7b3d3068ce %}

It was a success, I was able to connect and haven't been asked for a password.

![](https://miro.medium.com/0*8zAmW_3TqKQFDjdA)

Please note, the above process must be done for all nodes on cluster.

## Hosts file update

Finally, it's important to update the hosts file on each node by adding an entry of IP/alias key-value pairs for all other nodes. This step will prove useful later in RabbitMQ cluster setup.

The hosts file is located at /etc/hosts. I have to login to all nodes and update the file by adding an entry for all other nodes in cluster. For example, in rpi-node1 I've added the following.

{% gist 7ae1030db1728c3676454cb2665545fc %}

## Installing Erlang

RabbitMQ requires Erlang/OTP to run. Erlang is a functional programming language, designed to be a distributed, fault-tolerant system that guarantees 99.999% uptime. Erlang supports distributed systems and provides a very robust communication model, something that RabbitMQ takes full advantage for clustering, with servers making use of the inter-process communication (IPC) system to talk to each other.

I have to tell you that I faced some challenges in installing Erlang and RabbitMQ on the nodes. Mostly, the problems I faced had to do with versions of Erlang and RabbitMQ automatically inferred and, surprise, surprise, were not compatible on that debian image. For that reason, I decided to use the apt's [package pinning feature](https://wiki.debian.org/AptConfiguration?action=show&redirect=AptPreferences) in order to specify the version I want to be installed.

To pin erlang packages, I've created a preference in /etc/apt/preferences.d/erlang. For erlang and esl-erlang packages I set it to 22.1.6 as you can see below.

{% gist 55cd254ddcb8b5fe7df27850a425877f %}

Next, is RabbitMQ server, which is pinned on 3.8.7 version. Similarly, I have to create a rabbitmq preference in /etc/apt/preferences.d directory, so the full path will be /etc/apt/preferences.d/rabbitmq.

{% gist 381ef840797627c8da3d96ee0cef0c9f %}

With the above packages pinned, I return to the home directory and download the Erlang Solutions repository. The following includes the public key for apt-secure as well, which is handy, less commands for me to type. :D

{% gist 9215e35d91a74963aa7801ebe889fe8b %}

Awesome, what is left now is to install some Erlang dependencies and the actual esl-erlang package! The "esl-erlang" package is a file containing the complete installation, it includes the Erlang/OTP platform and all of its applications. The "erlang" package is a frontend to a number of smaller packages.

{% gist 7052fca7e32b608d8b6023a4d8a6c158 %}

For more information on Erlang versions and installation of Erlang, see Erlang Solutions [manual](https://www.erlang-solutions.com/resources/download.html).

To confirm that Erlang is installed run the erl command.

![](https://miro.medium.com/0*ergyLOzwUrTvLace)

If it is installed correctly, expect to see something similar to the following output

{% gist ed1f413936c1ea7b095ab6c1c0227d75 %}

## Setup Erlang cookie

Another important task is to set up the Erlang cookie, as is essential for the RabbitMQ cluster multi-node communication. The cookie file must have the same value across all servers in the cluster, otherwise they won't be able to communicate with each other. Before I install RabbitMQ and start the servers, I am going to create that file in its designated location. It's much better to do now, because if I do that after RabbitMQ runs for first time, I will then have to stop the service, update the file and then run the service again, which is a chore I'd like to avoid.

For reasons stated above, I've created an _.erlang.cookie_ file on the /var/lib/rabbitmq directory with a simple string value.

{% gist 89002d2e59951640b4f9337fd4986440 %}

The same string value, _QUERTY_, **must** be set to all cookie files across all server nodes.

## Installing RabbitMQ

RabbitMQ's biggest dependency is now installed, what is left is the actual message broker, but thankfully this is not complex task at all. First, few required dependencies need to be installed manually, with all listed below.

{% gist 158f9ee16602fc68fe7e6f2c522ddcfe %}

Next step is to install RabbitMQ by downloading the 3.8.7 version from GitHub and manually installing it via dpkg. To find a list with all RabbitMQ available [releases](https://github.com/rabbitmq/rabbitmq-server/releases) please check their [releases](https://github.com/rabbitmq/rabbitmq-server/releases) page.

{% gist c46e27b9801291fc628d5558346996a2 %}

For more information, see RabbitMQ's official docs on [manual installation](https://www.rabbitmq.com/install-debian.html#manual-installation) with [dpkg](https://man7.org/linux/man-pages/man1/dpkg.1.html).

## Running RabbitMQ

With RabbitMQ installed, I'm now ready to start the rabbitmq-server daemon and get RabbitMQ up and running. First, I have to enable the daemon by running the systemctl enable command and then start it by running the systemctl start command. The last command listed below is enabling the [RabbitMQ Management Portal](https://www.rabbitmq.com/management.html), which is very useful and I definitely want that in order to monitor my cluster health, connections, exchanges, queues, policies, etc. It comes as a plugin, so I only have to enable it as it seems.

{% gist 06768b689368e1ae744814868cad4a48 %}

### Accessing management portal in the local network

But how am I going to access the management portal if my RaspberryPi OS doesn't support desktop features? The answer to this question is quite simple, the same way I was able to control my Raspberry Pi from my Windows 10 PC. Remotely! However, the guest/guest credentials don't work outside the local machine, so I have to create a new user in order to access the management portal remotely. Below, I've included a snippet to create a new user ****__test__**** with password ****__test__**** (not very imaginative huh?) and assign it an administrator role.

{% gist 671e1188459e13f13c15756995579253 %}

With the above completed, I can now test if that works, by accessing the management portal from my PC. The master node is located at 192.168.0.24 on local network and the port for management portal is 15672.

![](https://miro.medium.com/0*RAJkn596Ho6nha1p)

I repeated the same steps for the rest of the nodes.

## Adding nodes to the cluster

The final step in the RabbitMQ cluster set up is to get the nodes join the actual cluster. In the following example, I'm going to make node1 join the cluster, however please keep in mind that the same process must be followed for the rest of the nodes, node2, node3 and node4.

First, I got to login to rpi-node1 from my PC and stop RabbitMQ. Then I have to erase the RabbitMQ runtime state in that node in order to successfully have it join the cluster.

{% gist 50369f83e542b13648ba01f71b12ee18 %}

With the rpi-node1 RabbitMQ stopped, I can now join it to the primary node, i.e. the master, to form the cluster.

{% gist e22450cd16beb1f8ef646b76487fce0f %}

Finally, the server must start again with the following command

{% gist ad707a314e2735598c7bdc08bdddd3a7 %}

The same process now should be followed on rpi-node2, rpi-node3 and rpi-node4.

## Reviewing cluster setup from management portal

That's how the management portal looks with all nodes added into the cluster.

![](https://miro.medium.com/0*QYB_zllHLKqmKIza)

## Summary

By now, I hope that you have learned how to setup a cluster in your local network and configure the nodes appropriately so they are able to communicate with each other. We've seen that the process of installing and configuring RabbitMQ's built-in clustering capabilities requires a bit of effort, especially if this is done manually as in the above examples, however I do hope that you now have more confidence in troubleshooting RabbitMQ and that you understand better how to set up your own cluster from scratch.

Possibilities are endless at this point, this cluster offers the opportunity for experimentation, like benchmarking publishers, consumers, using HA queues and many other things, which I will gladly share with you guys in future posts! Thanks for reading!

---

If you liked this blog, please like and share! For more, follow me on [Twitter](https://twitter.com/giorgosdyrra).