---
layout: post
title: Weave vs Calico
excerpt: Making a choice on a CNI (Container Network Interface, necessary for pod to pod communication) to use in production is not always easy and replacing a CNI after the fact is also not an easy task. Weave and Calico are one of the most popular CNIs out there and I have been lucky to run both of them in production, in this post I'll attempt to provide a non-bias review of both CNI implementations.
modified: 2018-05-21
tags: [kubernetes, calico, weave, cni, networking, containers]
comments: true
---

Making a choice on a
[CNI](https://github.com/containernetworking/cni#cni---the-container-network-interface)
(Container Network Interface, necessary for pod to pod communication) to use in
production is not always easy and replacing a CNI after the fact is also not an
easy task. Weave and Calico are one of the most popular CNIs out there and I
have been lucky to run both of them in production, in this post I'll attempt to
provide a non-bias review of both CNI implementations.

### Installation

#### Weave
Installing Weave is pretty straightforward, from the [official
docs](https://www.weave.works/docs/net/latest/kubernetes/kube-addon/#install):

{% highlight bash %}
$ kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
{% endhighlight %}

#### Calico
Installing Calico is more involved since it requires an etcd v2
cluster to function, you have to make a decision if you intend on using the
same cluster you use for Kubernetes or you want to set up an entirely different
etcd cluster for Calico. Once you download the [calico
configs](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/calico.yaml)
you can modify the `etcd_endpoints` as described in the [official
docs](https://docs.projectcalico.org/v2.6/getting-started/kubernetes/installation/hosted/hosted)
and then `kubectl apply calico_config` to install Calico.

### Protocol

#### Weave

Weave operates at layer 2 of the OSI layer and uses the VXLAN protocol to
overlay a layer 2 network on an existing network, Weave uses the Open vSwitch
kernel module to program the kernel's
[FDB](https://en.wikipedia.org/wiki/Forwarding_information_base) (Forwarding
Database). Weave only requires port 6783 (TCP and UDP) and port 6784 (UDP) open
on the nodes that need to participate in the overlay network. Pods on the
overlay network can then transparently communicate with each other like they
were all plugged into the same switch.

VXLAN is a relatively new protocol; the VXLAN RFC ([RFC
7348](https://tools.ietf.org/html/rfc7348)) was published in August 2014.

#### Calico

Calico operates at layer 3 of the OSI layer, it uses the BGP protocol to share
route information among the nodes participating in the network with each of
the node acting as a gateway to the pods running on them. Calico requires port
179 (TCP) open. Calico makes each of the node proxy_arp to the pods running on
each of them and installs a default gateway `169.254.1.1` on each of the pods,
e.g:

{% highlight bash %}
$ ip ro sh
default via 169.254.1.1 dev eth0 
169.254.1.1 dev eth0  scope link
{% endhighlight %} 
Inspecting the neighbors known to the pod reveals just one neighbor as below:

{% highlight bash %}
$ ip ne sh
169.254.1.1 dev eth0 lladdr fa:1a:23:4c:0a:c0 REACHABLE
{% endhighlight %}

That neighbor is the other end of the veth pair on the node, as shown below:

{% highlight bash %}
$ ip link sh cali23b602f82b4
29: cali23b602f82b4@docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default 
    link/ether fa:1a:23:4c:0a:c0 brd ff:ff:ff:ff:ff:ff
{% endhighlight %}

The BGP Protocol has been around for a while, the earliest RFC ([RFC
1654](https://tools.ietf.org/html/rfc1654)) for BGP was written in July 1994.
The BGP protocol powers a significant portion of the internet.

### IPAM (IP Address Management)

#### Weave

Weave uses a CRDT to fairly distribute IP addresses among nodes, it is well
documented
[here](https://github.com/weaveworks/weave/blob/master/docs/ipam.md), so I will
not rehash it.

#### Calico

Calico assigns a `/26` IP address block to each node and stores the assignment
in etcd, the IP blocks are bound to the nodename in etcd. When using an ASG one
needs to run `calicoctl delete node <nodeName>` when an instance is being
deleted or come up with tricks to ensure the nodename is consistent across
recreation, otherwise the entire pod cidr can be easily depleted. Weave had a
similar problem but it has been fixed in this
[PR](https://github.com/weaveworks/weave/pull/3022). ~~Assigning a `/26` can
lead to IP address fragmentation where some nodes still have free IP addresses
but other nodes completely run out of IP address blocks~~ **UPDATE** Calico
does not fragment IP address allocation, when the entire CIDR is exhausted and
there is no `/26` to assign, it will attempt to steal `/32` addresses from the
other `/26` blocks that have free IP address slots.

### Multi-Subnet Support

#### Weave

As long as the nodes participating in the cluster can reach each other on port
6783 (UDP and TCP) and port 6784 (UDP) the overlay will just work.

#### Calico

Calico works at Layer 3 and sets up each of the nodes as gateway to the pods
running on them. In standard networking a host requires layer 2 connectivity to
the gateway it configures for any route, due to this limitation BGP alone will
not work, however Calico has support for IPIP protocol for traffic crossing
layer 2 boundaries. For IPIP to work you need to allow the traffic through your
firewall.

### Network Debugging

#### Weave

Debugging packet traversal with tools like `traceroute` does not reflect nodes
the packet passed through before getting to the final pod, since the pod
assumes direct connectivity. To introspect information programmed in the kernel
with Open vSwitch you will have to use tools like
[odp](https://github.com/weaveworks/go-odp).  Debugging an overlay network
requires a different thought process from what most linux/network
administrators are used to since you have to be cognizant of the fact that
layer 2 becomes layer 3 and layer 3 becomes layer 2 when a packet goes from a
pod on one node to a pod on another node. Weave programs iptables only when
there are changes in the cluster, so you can modify the rules for debugging
purposes and be guaranteed they will stay the same if nothing changes in the
cluster with respect to the node while you are debugging.

#### Calico

Debugging packet traversal with tools like `traceroute` should indicate the
node the packet passed through before getting to the destination pod.
Introspecting routes installed can be done using the iproute2 utility e.g: `ip
ro sh proto bird table all`. Calico continuously attempts to keep the state of
iptables synchronized with its assumed internal state (a similar behavior to
kube-proxy) which could be frustrating when you are attempting to debug and it
installs its rules first so you can't easily log Information about the
iptables chains a packet traverses in order to debug iptables related problems.

### Encryption

Weave supports IPsec encryption out of the box while Calico does not support
any form of encryption.

### Network Policy

They both support ingress ~~and egress~~ Network Policy (Weave only supports
ingress and has an [open
issue](https://github.com/weaveworks/weave/issues/2624) on egress). Calico has
support for both egress and ingress and is the pioneer.

### Capacity

By default Calico and Weave are both full meshes so they both begin to degrade
once the cluster reaches a certain size (about 100 nodes). They both have
support for reducing the full mesh problem, Weave via [multi-hop
switching](https://www.weave.works/docs/net/latest/tasks/manage/multi-cloud-multi-hop/)
and Calico via [route
reflection](https://docs.projectcalico.org/v2.6/usage/routereflector/calico-routereflector).
