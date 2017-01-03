---
layout: post
title:  "Monitoring OpenShift for Service Providers"
date:   2017-01-03
categories: openshift monitoring
---

Many IT teams are deploying OpenShift as a self-service platform, changing their 
relationship with the teams responsible for developing, deploying, and 
maintaining software. Understanding that this is a very different model than many 
teams are used to, what should an IT team monitor to ensure that the platform is 
healthy?

A standard OpenShift deployment has a variety of information sources 
that can be used as input to monitoring and alerting tools. Before spending too
much time on what to monitor, it is helpful to understand what tools are in our
toolbox:

1. The OpenShift/Kubernetes API: Here is where we can get data about the
currently running pods and created resources, as well as watch for events.
2. Metrics/Hawkular: This is the system used internally to capture metrics from running
containers. Dependening on the version of OpenShift, this captures per container
CPU, memory, and network utilization. Hawkular has a full REST API (as used by
the OpenShift Web UI), so this data can be polled as part of a monitoring
solution.
3.  Logging/ElasticSearch: All logs from containers and OpenShift are aggregated
into ElasticSearch, usually utilized when using Kibana to perform ad-hoc
analysis of logs. But we can also submit our own queries to ElasticSearch as
part of a monitoring solution, or deploy a custom external Fluentd aggregator to
forward log data to our moniotring solution.
4. HAProxy: The default routing solution for OpenShift is HAProxy, meaning that
all external communication into OpenShift is reverse-proxied by HAProxy. HAProxy
provides a status endpoint that can be used to see which backends are returning
error codes or have poor response times. This endpoint isn't as polished as the
others and requires more work to get to.
5. Health checks: Several pieces of OpenShift have health checks that can be
queried to check system health. The most important ones are the master API,
etcd, routers, and ElasticSearch.
6. Diagnostics: The `oc` command line tool includes a diagnostic function that
can be executed with `oc adm diagnostics`, and can be run per node to check node
health and configuration. The tool spits out a fair bit of false positives, so
filtering is neccessary.

Monitoring Goals
----------------

Given those tools, what is it we want to achieve as a _service provider_ in our
monitoring solution? I blieve there are three main questions we want to answer:

1. Is the platform currently operational? Is there any reason a currently
deployed application would fail because of the platform instead of an internal
error or an incorrect deployment?
2. Is the platform in danger of not being operational in the near future? These
would be conditions that do not affect the current operations of OpenShift but
do leave it in a vulernable state if additional failures happen.
3. Does the platform have capacity for current and forecasted demand?

### Is the platform currently operational? (or Oh My... Fix It!) ###

Checking if the platform is currently up and running is relatively
straightforward. If any of these checks fails, there is an actionable situation
that requires _immediate attention_. For this we can mainly rely on our healthchecks:

* Are the masters healthy?

  Masters have a `/healthz` endpoint which will respond with a 200 status
code of the master is healthy. The body says "OK", as if the 200 status code
hadn't accomplished that already! Assuming this is an HA setup, use your load 
balanced public URL to ensure the public API is accessable (ie something like
`https://master.paas.example.com:8443/healthz`).

  If this check ever fails, it means that none of the masters are healthy or
that the load balancer in front of the masters is misconfigured. Either way,
users will not be able to use the CLI, UI, or API and no pods will be
scheduled, no deployments will proceed, etc. However, as long as nothing else
fails any deployed pods will continue to work. Either way, not a situation you
want to let persist for too long.

* Is the etcd cluster healthy?

  Etcd is the critical piece of infrastructure backing any HA dpeloyment of
OpenShift, and is one of the only components that actually has to deal with
clustering. Etcd is a quorum based CP data store. Most deployments will use
three etcd instances, meaning that the etcd cluster is healthy as long as two
of the three instances are healthy and connected.

  To check the health of etcd, you can either directly query the API or use a
CLI tool.

  To directly query the API, do an HTTP GET on each of the etcd hosts
(usually the masters) at `https://MASTERIP:2379/health` using the client cert
stored on a master at `/etc/origin/master/master.etcd-client.crt` and the key at
`/etc/origin/master/master.etcd-client.key`. A healthy response returns the JSON
`{"health": "true"}`. You should get this response from at least two out of
three etcd cluster members.

  To use the `etcdctl` CLI (installed on masters by default) execute the 
following, replacing the URLs with the etcd hosts:

   ```
   etcdctl -C \
     https://etcd1.example.com:2379,https://etcd2.example.com:2379,https://etcd3.example.com:2379 \
     --ca-file=/etc/origin/master/master.etcd-ca.crt \
     --cert-file=/etc/origin/master/master.etcd-client.crt \
     --key-file=/etc/origin/master/master.etcd-client.key cluster-health
   ```

  As long as the last line of output says "cluster is healthy", the etcd cluster
is operational. Note that one out of three cluster members may still report
being unhealthy.

* Are the routers healthy?

  Almost all traffic from external clients to hosted pods come in through the
routers, HAProxy instances that reverse proxy traffic to the wildcard DNS
address and forwards it to an appropriate pod. Routers also have a `/healthz`
endpoint, but it is on port 1936 and http (ie something like
`http://wildcard.paas.example.com:1936/healthz`). By default this port is blocked by
iptables, so a rule has to be added before using this endpoint on each node the
router may run on. Also, assuming an HA setup the load balancers in front of the
routers will need to also listen on this port.

  If this check fails, then no external traffic is making it into OpenShift and
anything deployed inside of OpenShift is down. Either all of the router pods are
down or the load balancer in front of the routers is misconfigured.

* Does `oc adm diagnostics` detect any issues?

  As mentioned above, the CLI includes a set of diagnostics that can be run from
each node to verify that the node is configured and functioning correctly. Each 
version of OpenShift adds more capabilities to the
diagnostics, so expect to make changes to any alerting based on diagnostics
after an upgrade.  The diagnostics also report a fair number of false positives
(like master nodes being marked unschedulable), so filtering is necessary.

* Are platform hosted applications (Kibana, Hawkular, Registry) responding
according to SLAs?

  There are a couple parts of the self-service platform that are actually hosted
_inside_ of OpenShift, just like any other application. There are two ways that
we can do a basic check that the application is alive and reachable.

  First, we can setup an external monitoring stack to do a health check of the
application. For example, a basic HTTP health check for
`https://kibana.paas.example.com` will let us know if kibana is responding
positively.

  Second, we can observe the response codes and latencies for requests coming from
our users. This helps us catch deeper issues where the service may be up but is
not responding to requests correctly, or is becoming too slow. These statistics
are tracked by the router (ie HAProxy). [Here are
instructions](https://docs.openshift.com/container-platform/3.3/admin_guide/router.html#viewing-statistics) 
for how to get access to these statistics. This will bring you to an easy to
read web page, but to get machine readable output just tack on `;csv` to the
url. What we are looking for is a large spike in 500 error codes for each of the
hosted applications, or an increase in response times that breaks any defined
SLA.

### Is the platform in danger of not being operational in the near future? (Or You Probably Should Take a Look at This) ###

These checks are a second pass that will detect potential issues that could lead
to downtime before downtime actually happens. Alerts should only be generated
after multiple failed checks, and depending on SLAs can wait to be acted upon
until normal business hours.

* Degraged Healthchecks

  For these alerts, repeat the the same healthchecks for masters, etcd, and
routers. Instead of checking for overall health though, fail the check if any
single instance has failed. Instead of checking the masters through the load
balancer, check each of the three masters directly by hostname. Do the same for
routers. For etcd, fail the check if `etcdctl` does not report any of the three
cluster members as healthy (a healthy result here looks like `member
fd422379fda50e48 is healthy: got healthy result from http://127.0.0.1:32379`).

  To keep restarts an other ephemeral failures from causing lots of alert noise,
ensure that these alerts only get sent after a period of failure. For example,
only alert if a master has been down for at least ten minutes. Also consider
whether or not these alerts can wait for action based on expected recovery time,
exposure to risk of an additional failure causing downtime, and SLAs.

* Registry Storage

  OpenShift builds images on request or when triggered by changes to application
source code. The images are then stored inside OpenShift's internal registry,
which uses persistent storage setup as part of the installation process. Docker
images are massive, and can quickly add up through repeated builds. Monitor the
space available in the persistent volume and alert when it is low.

  The main method for cleaning up the registry is to setup a [recurring job
(usually with cron) to execute `oc adm prune images
...`](https://docs.openshift.com/container-platform/3.3/admin_guide/pruning_resources.html#pruning-images).
If still low on space after pruning, then either adjust the prune to be more
aggressive (ie keeping less versions of each image or for less time)` or
allocate more storage to the registry.

  Running out of registry storage will not affect any currently running pods, only
the ability to build or push new images.

* Docker Storage

  Each node has a local docker daemon, and if following best practices it was
setup with a LVM logical volume for docker storage. Any time a container is
deployed the corresponding image is saved into docker storage and everytime a
container is started the container filesystem is in this storage. With lots of 
deployments this storage can quickly be filled.

  Monitor the space available on each node for Docker storage. If a node runs
out of space, then newly created pods will fail. OpenShift automatically garbage
collects docker storage, but the defaults may be too lax depending on usage
patterns. If running out of storage, [change the configuration for garbage
collection](https://docs.openshift.com/container-platform/3.3/admin_guide/garbage_collection.html)
to be more aggressive.

### Does the platform have capacity for current and forecasted demand? ###

These checks are focused on the current utilization of OpenShift and providing
information for capacity planning. Given reliable forecasts, these metrics
should rarely need to be alerted on.

* Master CPU and Memory Utilization

  Master utilization is linearly correlated with the number of pods deployed,
and [guidelines for sizing masters are
documented](https://docs.openshift.com/container-platform/3.3/install_config/install/prerequisites.html#production-level-hardware-requirements).
If masters become overloaded expect operations like scheduling pods to be
slower, and in the worst case for etcd to begin having timeouts in its cluster
communication.

* Router Node CPU Utilization

  There are not yet guidelines for router node sizing, but a reasonable
expectation would be that the CPU utilization would be linearly correlated with
incoming requests. If CPU utilization keeps hitting a high threshold then it may 
be time to scale the router out to an additional node.

* Metrics and Logging Storage

  Both metrics and log aggregation require persistent storage. 

  [Metric storage
utilization](https://docs.openshift.com/container-platform/3.3/install_config/cluster_metrics.html#capacity-planning-for-openshift-metrics) 
is very predictable given the number of pods, retention period, and metric
resolution. 

  Logs are much more difficult, since storage requirements are highly
dependent on how much data is logged by the application and log retention
policies as specified by [curator
configuration](https://docs.openshift.com/container-platform/3.3/install_config/aggregate_logging.html#configuring-curator).
A misbehaving application can quickly generate gigabytes of logs, so maintain a
healthy margin of available storage and alert fairly early so that storage can
be added or the application stopped or modified. 

* Node Resource Allocation

  Most service providers will want to control resource utilization by their
clients by setting quotas for resource utilization. For example, a project might
be limited to using 4 CPU cores and 10 GiB of memory. When using quotas, each
container must also have resources requests and/or limits configured, either via
explicit requests/limits in the pod specification or via default project limits.

  This is where things get tricky: when the Openshift scheduler places a pod it
is not looking at current utilization of the node to determine if the pod fits,
it is looking at the sum of all resource requests made by containers on the
node. As long as it fits within the project's quota, a container can request 2
GiB of memory but use none of it. As a Service Provider, you need to monitor
resources you have _allocated_ to clients, not the _current_ utilization of
nodes.

  Exactly what needs to be monitored depends on how node's have been partioned.
For example, if nodes are set aside for "production", "staging", and
"development", then allocations need to be monitored for all three seperately.
This is easier if projects are aligned with nodes, ie if there are "development"
nodes then there is a "development" project for deploying to those nodes, 
enforced via a project level node selector.

  There are two different ways to monitor allocations. The conservative approach
would be to monitor project quotas, assume that a certain percentage (may be
100%) of all project quotas are in use, and ensure enough node resources to
satisfy those requirements with a margin. The less conservative approach is to
monitor the currently used portion of each quota and add a margin for
expected new deployments, scale out, etc in the near future.

  To find all project quotas, a cluster administrator can run `oc get quota
--all-namespaces`, but by default this doesn't print the current quota values.
The following command adds a template to print out the assigned quota for each
project as well as how much of the quota is currently in use. Sum the first two
columns for a conservative capacity target and the last two colums for the
currently allocated capacity.

{% raw %}
  ```
  echo "cpu_limit memory_limit cpu_used memory_used" && \
    TEMPLATE='{{define "line"}}{{index . "requests.cpu"}} ' && \
    TEMPLATE+='{{index . "requests.memory"}}{{end}}' && \
    TEMPLATE+='{{range .items}}' && \
    TEMPLATE+='{{template "line" .status.hard}} ' && \
    TEMPLATE+='{{template "line" .status.used}}' && \
    TEMPLATE+='{{end}}' && \
    oc get quota --all-namespaces --template="$TEMPLATE" && \
    oc get clusterresourcequota --all-namespaces --template="$TEMPLATE"
  ```
{% endraw %}

### About OpenShift ###

For anyone unfamiliar with OpenShift, it lets you deploy an internally hosted 
Platform as a Service (PaaS) built on top of Kubernetes and adding features like a 
blessed set of components, automated builds, deployments, more powerful security 
controls, etc. You can read more about what OpenShift is 
[here](https://www.openshift.com/container-platform/features.html), sign up for
a 30 day [hosted preview](https://www.openshift.com/devpreview/), or download an 
[all-in-one VM](https://www.openshift.org/vm/) with everything already deployed.

 
