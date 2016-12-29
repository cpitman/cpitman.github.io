---
layout: post
title:  "TCP Servers and OpenShift"
date:   2016-12-28
categories: openshift tcp networking
---

When migrating systems into OpenShift it isn't uncommon to run into applications that use TCP protocols (that are not HTTP or HTTPS). If the TCP protocol is only used within OpenShift (ie between two containers hosted in OpenShift), then we don't need to do anything special because both containers are part of the same software defined network (SDN). Things become trickier when we want an external application (ie not on OpenShift) to communicate with our TCP server hosted in an OpenShift container.

OpenShift has a couple of built-in features to make this possible:

1. If the TCP protocol uses TLS, and the clients support Server Name Indication (SNI), then we can configure a TLS Passthrough route that sends all traffic through the OpenShift Router. Passthrough routes mean that the data is never decrypted, and we don't care what the protocol is (as long as it is over TLS). SNI is a TLS feature where clients can send the unencrypted hostname as part of the initial connection to the server, letting load balancers/virtual servers route traffic to the correct backend without decrypting the traffic. Most [browsers have supported SNI for HTTPS][1] for a long time.

   Support by other protocols and clients is on a case-by-case basis. The two ways I use to determine of a protocol supports SNI are (1) Google it or (2) use wireshark to inspect a client connecting to a server.
   
   In cases where the protocol supports SNI, this is by far the easiest/best technique to use.
   
2. If the protocol you are working with doesn't use TLS or your clients cannot support SNI, there are two built-in alternatives (one if using older versions). If you are using OpenShift Origin <=1.2 or Enterprise <=3.2, then you can use a [NodePort Service][2]. A NodePort service asks OpenShift to allocate a unique port to your service (say port 5005). Once allocated, every OpenShift node will listen for traffic on port 5005 and forward it to your service.

   Each port has to be unique across the cluster, so using NodePort means configuring clients after deploying the service to OpenShift in order to find an available port. Care is also required to make sure that the newly allocated port is not blocked by any firewalls between the client and server. Finally, if this service needs to be HA, it is a good idea to configure a load balancer to balance traffic across multiple nodes for redundancy.
   
3. Starting in OpenShift Origin 1.3 and Enterprise 3.3, an additional mechanism was added called ExternalIPs. When a service requests an External IP, it is assigned a non-sdn IP address from a pool. Assuming that the network is properly configured to route traffic for the External IP to a node, then OpenShift will forward all traffic to that IP to the service. Making this HA then requires additional setup of an IP Failover router. Red Hat Consulting has a good write up on how to do this in its [OpenShift Playbooks][3].

   ExternalIPs are a good solution given (1) you have an up to date OpenShift cluster and (2) have support from a strong networking team that will understand the required setup.
   
Given that the first solution involves no additional networking setup, is there a way that we can make more protocols work over the OpenShift Router? You bet! In my [next post]({% post_url 2016-12-28-stunnel-and-openshift %}), I'll layout how you can use stunnel to bridge arbitrary TCP traffic into OpenShift without any network changes and with the added benefit of securing/encrypting all traffic. 


[1]: https://www.digicert.com/ssl-support/apache-secure-multiple-sites-sni.htm
[2]: http://kubernetes.io/docs/user-guide/services/#type-nodeport
[3]: http://playbooks-rhtconsulting.rhcloud.com/playbooks/operationalizing/ingress.html
