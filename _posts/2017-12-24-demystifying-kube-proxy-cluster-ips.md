---
layout: post
title: Demystifying Kube Proxy (Cluster IPs)
excerpt: Cluster IPs (sometimes referred to as service IPs) are usually described with the term "magic", in this post I will attempt to remove the magical veil on Cluster IPs explaining how kube-proxy creates them and how they work.
modified: 2017-12-26
tags: [kubernetes, networking, containers, cluster-ips, services, service-ip, kube-proxy]
published: true
comments: true
---

Cluster IPs (sometimes referred to as service IPs) are usually described with
the term "magic". In this post I will attempt to remove the magical veil on
Cluster IPs explaining how kube-proxy creates them and how they work.

Every Kubernetes installation has a `kubernetes` service which is assigned a
cluster IP (usually the first available IP address in the cluster cidr), for
example:

{% highlight bash %}
kubectl get svc -n default
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   192.168.255.1   <none>        443/TCP   46d
kubectl get ep -n default
NAME         ENDPOINTS                                            AGE
kubernetes   172.16.0.11:6443,172.16.0.12:6443,172.16.0.13:6443   46d
{% endhighlight %}

`192.168.255.1` in this case is the cluster IP for the `kubernetes` service
which points at the pool of kube-api-server endpoints
(`172.16.0.11:6443,172.16.0.12:6443,172.16.0.13:6443`). The immediate thought
that comes to mind is to hop on a node and attempt to ping the IP, so we try as
below:

{% highlight bash %}
ping 192.168.255.1 -c 2 -w 1
PING 192.168.255.1 (192.168.255.1) 56(84) bytes of data.

--- 192.168.255.1 ping statistics ---
1 packets transmitted, 0 received, 100% packet loss, time 0ms
{% endhighlight %}

Surprise! Pinging the IP does not work. How about curl

{% highlight bash %}
curl -kI https://192.168.255.1
HTTP/1.1 403 Forbidden
Audit-Id: e53f57e9-fc23-41a0-99b4-da02d2bacb5a
Content-Type: application/json
X-Content-Type-Options: nosniff
Date: Sun, 24 Dec 2017 16:35:02 GMT
Content-Length: 234
{% endhighlight %}

Seemed to have worked. To understand why curl worked and why ping did not work,
there has to be some form of nat'ing going on, so let's log the
iptables rules the packet traversed.

Iptables provides a `TRACE` target that can be used to log tables, chains and
rules a packet traversed, but before we can use the target we need to enable
and configure the logging backend for iptables `TRACE`.

{% highlight bash %}
$ modprobe nf_log_ipv4
$ sysctl net.netfilter.nf_log.2=nf_log_ipv4
{% endhighlight %}

We can easily write rules to match icmp requests and requests to port 6443 and
port 443 however that might end up being too expensive as so many pods on the
host can be making requests to the api-server and logging too many packets will
result in a performance penalty, so we can just spin up a dummy nginx pod for
this test as below:

{% highlight bash %}
$ kubectl run nginx --image nginx
$ kubectl exec -it nginx-4217019353-4hwq2 -- /bin/bash -c 'apt update && apt install -y --force-yes curl iputils-ping' # Installs curl and ping
{% endhighlight %}

We need to retrieve the IP address of the pod as we'll need it to write the
targeted iptables rules for logging, we can retrieve it as below:

{% highlight bash %}
$ kubectl get po -l "run=nginx" -o jsonpath="{.items[0].status.podIP}"
192.168.7.163
{% endhighlight %}

On the node the pod has been assigned, which can be retrieved with: `kubectl
get po -l "run=nginx" -o jsonpath="{.items[0].spec.nodeName}"`, we apply the
below rules:

{% highlight bash %}
sudo iptables -A PREROUTING -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -J TRACE
sudo iptables -A PREROUTING -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -j TRACE
sudo iptables -A OUTPUT -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -j TRACE
sudo iptables -A OUTPUT -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -j TRACE
sudo iptables -A OUTPUT -s 192.168.7.163 -p icmp -m icmp --icmp-type 0 -t raw -j TRACE
sudo iptables -A PREROUTING -s 192.168.7.163 -p icmp -m icmp --icmp-type 8 -t raw -j TRACE
{% endhighlight %}

These rules means TCP packets from the pod's IP address 192.168.7.163 going to
port 6443 or port 443 and ICMP echo request (ping) should be logged.

We can now go ahead to curl the service ip from the pod as below:

{% highlight bash %}
$ kubectl exec -it nginx-4217019353-4hwq2 -- curl -Ik https://192.168.255.1
HTTP/2 403
audit-id: 87e719ca-01d2-4b45-a36e-e647fb59a0eb
content-type: application/json
x-content-type-options: nosniff
content-length: 234
date: Tue, 26 Dec 2017 01:23:47 GMT
{% endhighlight %}

If we go the node we should find entries similar to the below in either
`/var/log/kern.log` or `/var/log/syslog` (depending on your system
configuration).

{% gist d44dce927be66f71e45a959fcb57b41e %}

To narrow it down as it is in the log you can pick any ID and just grep for it
e.g `40679` in this case. Looking at the `DST` and `DPT` fields of the log we
observe that `DST` changes from `192.168.255.1` to `172.16.0.11` and `DPT` from
`443` to `6443` after this line:

```
Dec 26 01:09:22 nodename kernel: [1220127.957926] TRACE: nat:KUBE-SEP-23Y66C2VAJ3WDEMI:rule:2 IN=cali23b602f82b4 OUT= MAC=fa:1a:23:4c:0a:c0:6e:64:e6:2b:cb:40:08:00 SRC=192.168.7.163 DST=192.168.255.1 LEN=60 TOS=0x00 PREC=0x00 TTL=64 ID=40679 DF PROTO=TCP SPT=47762 DPT=443 SEQ=3688814824 ACK=0 WINDOW=29200 RES=0x00 SYN URGP=0 OPT (020405B40402080A122D460F0000000001030307) MARK=0x4000000 
```

So there has to be something special about rule 2 of the
`KUBE-SEP-23Y66C2VAJ3WDEMI` (the chain name will be likely be different in your
setup because the `23Y66C2VAJ3WDEMI` part of the name is randomly generated)
chain on the `nat` table, we can determine the rules matching that chain by
grepping against the internal state of iptables, as below:

{% highlight bash %}
$ iptables-save | grep KUBE-SEP-23Y66C2VAJ3WDEMI
:KUBE-SEP-23Y66C2VAJ3WDEMI - [0:0]
-A KUBE-SEP-23Y66C2VAJ3WDEMI -s 172.16.0.11/32 -m comment --comment "default/kubernetes:https" -j KUBE-MARK-MASQ
-A KUBE-SEP-23Y66C2VAJ3WDEMI -p tcp -m comment --comment "default/kubernetes:https" -m recent --set --name KUBE-SEP-23Y66C2VAJ3WDEMI --mask 255.255.255.255 --rsource -m tcp -j DNAT --to-destination 172.16.0.11:6443
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-23Y66C2VAJ3WDEMI --mask 255.255.255.255 --rsource -j KUBE-SEP-23Y66C2VAJ3WDEMI
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-23Y66C2VAJ3WDEMI
{% endhighlight %}

The second rule indicates it's being DNAT'ed to `172.16.0.11` port `6443`, it
also uses the `recent` (Allows you to dynamically create a list of IP addresses
and then match against that list in a few different ways) module to specify via
the `--rsource` flag that iptables (netfilter) should save the source address
and match against it. In combination with the `--rcheck` `--seconds 10800
--reap` on the next line iptables will ensure all requests in a `10800` seconds
window from the same source IP will be sent to the same kube-api-server, this
is configured via `service.spec.sessionAffinityConfig.clientIP.timeoutSeconds`
in the service spec.

The last rule looks very interesting, the chain `KUBE-SVC-NPX46M4PTMTKRN6Y`
seems to appear twice in the output, it also came before the
`KUBE-SEP-23Y66C2VAJ3WDEMI` chain in the logs, let's see what rules have the
chain in them, we can for the internal state of iptables (netfilter) as below:

{% highlight bash %}
$ iptables-save | grep KUBE-SVC-NPX46M4PTMTKRN6Y
:KUBE-SVC-NPX46M4PTMTKRN6Y - [0:0]
-A KUBE-SERVICES -d 192.168.255.1/32 -p tcp -m comment --comment "default/kubernetes:https cluster IP" -m tcp --dport 443 -j KUBE-SVC-NPX46M4PTMTKRN6Y
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-23Y66C2VAJ3WDEMI --mask 255.255.255.255 --rsource -j KUBE-SEP-23Y66C2VAJ3WDEMI
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-47TYMEKETDMXSVAN --mask 255.255.255.255 --rsource -j KUBE-SEP-47TYMEKETDMXSVAN
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m recent --rcheck --seconds 10800 --reap --name KUBE-SEP-CZKGMRCAIW3ENCXY --mask 255.255.255.255 --rsource -j KUBE-SEP-CZKGMRCAIW3ENCXY
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m statistic --mode random --probability 0.33332999982 -j KUBE-SEP-23Y66C2VAJ3WDEMI
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -m statistic --mode random --probability 0.50000000000 -j KUBE-SEP-47TYMEKETDMXSVAN
-A KUBE-SVC-NPX46M4PTMTKRN6Y -m comment --comment "default/kubernetes:https" -j KUBE-SEP-CZKGMRCAIW3ENCXY
{% endhighlight %}

The 2 lines before the last uses the `statistic` (This module matches packets
based on some statistic condition) module to match packets based on some
probability, so when a packet is traversing this chain for a probability of a
third (`0.33332999982`) the packet is sent to the `KUBE-SEP-23Y66C2VAJ3WDEMI`
chain which we had previously looked at and then DNAT'ed to `172.16.0.11:6443`,
if a packet does not match the gets to the next line where based on a
probability of a half (0.5) it's sent to the `KUBE-SEP-47TYMEKETDMXSVAN` chain
which DNATs to `172.16.0.12:6443`, any packet not matching that probability is
sent to the `KUBE-SEP-CZKGMRCAIW3ENCXY` chain which DNATs it to
`172.16.0.13:6443`. This is how kube-proxy uses iptables to achieve a round-robin
load-balancing. The logic computing the probabilities can be found
[here](https://github.com/kubernetes/kubernetes/blob/980a5e80b12b35cb724aa276b8b589b5e96dd80f/pkg/proxy/iptables/proxier.go#L617-L637).

Given the rules are explicit to DNAT tcp requests from the cluster IP on port
443 to the kube-apiservers on 6443, ping (ICMP echo request) packets will leave
the host and the router will return with a host unreachable because the cluster
IP is not routable, which explains why ping did not work previously.

To clean up the rules we had previously added, replacing `192.168.7.163` with
your pod IP do:

{% highlight bash %}
sudo iptables -D PREROUTING -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -J TRACE
sudo iptables -D PREROUTING -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -j TRACE
sudo iptables -D OUTPUT -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -j TRACE
sudo iptables -D OUTPUT -s 192.168.7.163 -p tcp -m tcp -m multiport --dports 6443,443 -t raw -j TRACE
sudo iptables -D OUTPUT -s 192.168.7.163 -p icmp -m icmp --icmp-type 0 -t raw -j TRACE
sudo iptables -D PREROUTING -s 192.168.7.163 -p icmp -m icmp --icmp-type 8 -t raw -j TRACE
{% endhighlight %}
