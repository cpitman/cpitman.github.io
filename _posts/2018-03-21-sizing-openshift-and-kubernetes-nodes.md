---
layout: post
title:  "Optimally Sizing OpenShift and Kubernetes Nodes"
date:   2018-03-21
categories: openshift 
---

This should be a short topic: how do you decide how big to make nodes in an
OpenShift or Kubernetes deployment?

We have two main drivers for this decision: redundancy and efficiency. We need 
a minimum number of nodes to handle failures, and large enough nodes that our pods can be 
efficiently stacked to reduce wasted resources.

We'll size our nodes based on **Total Core Usage (TCU)**, the total number of CPU Cores that are 
required for all cluster workloads, and **Average Pod Cores (APC)**, the average
number of cores used by a single pod in the cluster:

1. (TCU < 50 cores and APC < 0.2 cores) OR (TCU < 100 and APC > 0.2 cores)

    At this size you'll want to have at least 3 app nodes for redundancy, and plan for one node to be down at any point in time. That means each node needs to have at least TCU/2 cores. Round up to a reasonable size to give a little headroom.

    * `Node Size ~= TCU/2`
    * `Node Count >= 3`

2. TCU > 50 cores and APC < 0.2 cores

    At this size redundancy is pretty much guaranteed because of scaling out.
    But your pods are small, and a node needs to be sized to prevent too many 
    pods being scheduled to a single node.

    You'll want to run fewer than ~250 pods per node (as of Kubernetes 1.9 and
    OpenShift 3.9). We also want to avoid wasting resources because no more 
    pods can be scheduled to a node, so size nodes for about ~200 pods.

    * `Node Size ~= 200 * APC`
    * `Node Count >= (TCU / Node Size) + 1`

3. TCU > 100 cores and APC > 0.2 cores

    At this size your workloads will fully consume any reasonable node size you
    throw at them. Your main concern should be getting the best deal you can on
    capacity. Pick whatever node size minimizes the dollars spent per core of
    capacity.

    * `Node Size = Minimize ($/core)`
    * `Node Count >= (TCU / Node Size) + 1`


Also, don't assume you can only choose one node size! If you have a few majorly
different kinds of workloads (like CPU intensive servives and memory intensive 
databases) you can create nodes sized for the different workloads. Apply labels
to the nodes and pods to help the scheduler efficiently place them.

Examples
--------

For the following examples I'll assume we are deploying to AWS, and that our 
workloads match the ["General Purpose"](https://aws.amazon.com/ec2/instance-types/) profile of 4 GBs of memory per core.

1. 40 Pods, each needing half a core:
  * TCU = 20
  * APC = .5
  * This is a small cluster, so we want at least TCU/2 cores per node, or 10 cores
  * We can either round down and get 4 m5.2xlarge nodes (8 cores) or 
  round up and get 3 m5.4xlarges (16 cores). Based on current pricing,
  4 m5.2xlarge nodes is more cost efficient.

2. 2000 Pods, each needing a tenth of a core:
  * TCU = 200
  * APC = .1
  * This is a large cluster with small pods. Node Size ~= 200*APC = 20 cores
  * m5.12xlarge (48 cores) is way too large for such small pods, so we'll use m5.4xlarge (16 cores)
  * To get the number of nodes, `200/16 = 12.5`. Round up and add at least one
  for failover to get 14 nodes.
    
3. 1000 Pods, each needing 2 cores
  * TCU = 2000
  * APC = 2
  * This is a large cluster with large pods. Pick the most cost efficient node
  size in terms of $/core. Based on current AWS pricing, all m5 VMs have the 
  same cost per core, so we'll pick the largest to reduce waste: m5.24xlarge (96 cores)
  * To get the number of nodes, `2000/96 = 20.8`. Again, round up and add at 
  least one for failover to get 22 nodes. At this number of nodes, it may make
  sense to add a couple more for additional failover capacity.


PS: Here is a [Kelsey Hightower talk](https://youtu.be/HlAXp0-M6SY?t=559) that
introduces the Kubernetes scheduler and the problem it has to solve to 
efficiently use node resources.