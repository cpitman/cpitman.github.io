---
published: false
layout: post
title: OpenShift Quotas and Chargeback Models
tags: openshift operations
---
Any organization deploying a PaaS like OpenShift (or even IaaS, either public or private) has to figure out how to control resource utilization while also paying for the platform. Otherwise the result is a free for all, where users can consume all the resources that have been provisioned without any justification. At this point, you can either continuously expand the platform (skyrocketing costs) or accept a highly oversubscribed service that begins to buckle under load.

Controlling Resource Utilization
--------------------------------

First up, how do we control the resource utilization of users? As a service provider, the goal should be to provide flexibility while also incentivizing efficient use of resources.

The way we do this in OpenShift is to set a [Quota](https://docs.openshift.com/container-platform/3.4/dev_guide/compute_resources.html) that sets maximums for the resources used by an OpenShift project. There are quite a few resources that can be constrained, but for this topic the most important are CPU, Memory, and Storage.

For both CPU and Memory every pod can define a "request" and a "limit". A "request" guarentees a pod at least the defined amount of a resource. A "limit" constrols the maximum amount of a resource that can be used (ie a limit to bursting). So if a pod sets a CPU request of a half core (500mi) and a limit of 1 core (1000mi), then OpenShift will ensure the pod receives at least half a core at all times and let it use up to one core _if it is available_.

A quota can be set for each project to control the max amount of resources that can be assigned to all pods in the project. Just like with pods, a quota can set a cap on both requests and limits. If a quota has a CPU request of 2 cores (2000mi) and limit of 3 cores (3000mi), then all pods in the project must have requests and limits that sum up to less than the quota. Any pods created above the quota will be stuck waiting for resources to free up.

*NOTE:* Quotas are all about the resources a pod _could_ use, not the resources actually being used by a pod. In other words, if a pod requests 1 core (1000mi) but does nothing with the core, the core is still reserved for the pod and no other pod can use it. That is why it is important to incentivize users to efficiently allocate resources!

There is a newer way to specify quotas called [Multi-Project Quotas](https://docs.openshift.com/container-platform/3.4/admin_guide/multiproject_quota.html) that lets administrators specify a quota that applies to multiple projects based on some criteria. The most common example is specifying a quota for all projects created by a specific user, letting the user allocate resources across their projects as long as they stay under the quota. This lets users create their own projects but still constrains their resource usage. Again, as a service provider, the goal is to enable users! This feature was added in OpenShift 3.3.

Capacity Planning with Quotas
-----------------------------

Given how quotas work, what is the best way to plan our overall PaaS capacity? Ideally the PaaS would always have capacity added at exactly the right time, so that all resources are doing productive work and no productive work is unschedulable. In practice, this is difficult to do (unless you can burst out to a public cloud!).



On the opposite end, the PaaS can always be provisioned with enough capacity for the maximum amount of resources that could be requested. In other words, ensure that the PaaS has at least as many resources as the sum of all quotas currently allocated to users, plus additional capacity for the expected growth of allocated quotas. This is a very conservative approach, with a high chance that all workloads can be scheduled but also a fair bit of capacity that isn't utilized.

Chargeback and Incentivizing Efficiency
---------------------------------------
