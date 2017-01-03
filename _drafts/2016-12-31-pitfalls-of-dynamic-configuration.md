---
layout: post
title:  "The Pitfalls of Dynamic Configuration"
date:   2016-12-31
categories: ha architecture
---

Hey everyone! Today I'd like to tell you a story of "high availability"
run amuck, leading to lots of wasted time and effort with little to show
for it.

Several years ago I architected a large distributed messaging system that
had extreme HA requirements (99.999% uptime). Because of the HA requirement,
every component was deployed with many replicas and we could survive many
many failures without seeing any downtime. Regardless of this, we also
implemented a dynamic [TBD]

One of the most destructive lines of thought for system availability is 
"this system should be highly available, therefore it (and its components)
must never go down". This leads to more complex systems, which increases
the number of possible failure modes. It also focuses the team more on 
single-instance durability than surviving and recovering from failures.

A lot of this mindset is a hold over from having big monoliths that have
to be protected. If all of our business relies on a single instance of an
application, then downtime for that application is downtime for the business.
But these days most applications should be built to horizontally scale, so
that failure of any one instance doesn't _have_ to mean downtime.

What I am calling "Dynamic Configuration" is a set of features you can 
implement to let administrators reconfigure the application without requiring
the application be restarted. The specific implementation can have many 
variations, like file or database watching, an API, a UI, etc. But at it's
core, it requires implementing the ability to (1) detect changes to
configuration and (2) apply those configuration changes to currently running
processes/state. This is on top of the normal ability to read and apply 
configuration at startup.

Dynamic Configuration is a feature, but it is a very expensive feature to
implement and the cost scales with the amount of configuration that can be
modified at runtime. 

Some configuration can very easily be modified because it is read each time 
it would have an effect. The common example would be changing the level of
logging at runtime, since the choice of whether or not to log is evaluated
each time a log statement is executed.

On the other side is configuration that is difficult to modify. An example
would be changing a value that effects how connections to other components
are created. This could require killing and reconnecting to these components,
or an addition to the protocol in question to allow dynamic renegotiation of
configuration.

Whether easy to modify or hard to modify, each configuration value that can 
be dynamically configured requires extra planning, implementation, and testing
compared to static configuration at startup. If we run N instances if an 
application, then we don't risk downtime by restarting to update configuration.
And restarting to update configuration doesn't require any additional 
development time. Therefore, _most_ dynamic configuration is a waste of time
and resources.

So when is dynamic configuration not a waste? I'd argue the only time is when
restarting would lose state that *cannot* be saved. For example, changing log
levels to get more information from a malfunctioning application. Restarting
runs the risk of "fixing" the problem. Luckily, this is also the easier kind 
of configuration to modify at runtime!

Focusing on recovering from and surviving component failures makes dynamic
configuration (mostly) a thing of the past. Your time is better spent elsewhere,
like testing your recovery procedures!
