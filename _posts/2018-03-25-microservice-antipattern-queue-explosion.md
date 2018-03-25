---
layout: post
title:  "Microservice Antipatterns: The Queue Explosion"
date:   2018-03-25
categories: microservices 
---

Scenario
========

![Scenario Diagram]({{site.baseurl}}/images/queue_explosion_situation.png)

A microservice based architecture is being designed. Some of these 
microservices will need to interact with message queues in order to get 
asynchronous guaranteed processing. Often the requirement to use queues comes
from integrating with an external queue based system.

Definition
==========

Because queues must be used for *some* communication, they are used for nearly
all communication. This seems like a good solution since it reuses the same
technology and patterns across all the microservices.

![Definition Diagram]({{site.baseurl}}/images/queue_explosion_definition.png)

To be clear, this scenario relies not just on messaging but on *queues*. 
Messages are sent to another server, which must persist the message to reliable
storage before responding to the client or forwarding the message along.

Possible ill-effects include:

* Multiplying the load on message brokers, often 2-5x, requiring extensive and
  costly scale out of the messaging system
* No easy flow-control, fast services can produce and consume messages quickly
  enough to cause side effects for the slow services that are the bottlenecks and
  dictate overall system performance. This can cause large swings in 
  performance as queues fill up.
* Increased latency for end-to-end processing because of repeated round trips 
  to the messaging broker, which must persist messages to disk each time
* Increased difficulty isolating performance issues to an individual service vs
  the messaging brokers vs side-effects from other services impacting the 
  messaging brokers

Solutions
=========

Replace many (but not necessarily all) of the messaging queues with other 
communication:

* HTTP-based services (REST, [gRPC](https://grpc.io/), etc) offer low latency and high throughput
  compared to queues. They also have strictly lower resource requirements
  (network, disk, compute, memory) than a message queue solution.
* Messaging-based peer to peer or routed protocols allow services to follow
  similar messaging conventions, but instead of sending messages to a broker
  which persists messages, messages are either sent directly to another 
  consumer via a point-to-point network connection or are sent to a router
  which immediately sends the message to a consumer. Examples include 
  [ZeroMQ](http://zeromq.org/) and 
  [Apache QPID Proton](https://qpid.apache.org/proton/index.html).


![Solution Diagram]({{site.baseurl}}/images/queue_explosion_solution.png)

Microservices get most of the benefits of queues with sparing application. In
the above diagram the system can still smooth out spikes in requests with the
single queue, and the external system and microservices are still in isolated
fault domains. 

In return for using HTTP or similar protocols internally, the microservice
deployment will have a smaller footprint and be significantly easier to tune and
diagnose issues with.

In Summary
==========

Justify the use of each queue in a microservice architecture based on the 
individual needs to the participating microservices, not in an effort to use only
one technique for communication across an entire system. The correct blend of 
asynchronous and synchronous communication increases performance, reduces costs,
and simplifies operations and troubleshooting.

