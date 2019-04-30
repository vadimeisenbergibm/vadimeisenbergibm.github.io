---
title: Istio as a transparent DNS-aware Kubernetes-aware Egress Firewall
subtitle: Secure Egress Traffic Control in Istio
publishdate: 2019-04-30
attribution: Vadim Eisenberg (IBM)
keywords: [traffic-management,egress]
---

In this blog post I describe how Istio can be used for secure egress traffic control. While the most important security
aspect for a service mesh is ingress traffic to prevent attackers from penetrating the cluster though ingress APIs,
securing the traffic leaving the mesh is also very important. Once your cluster is compromised, and you must be
prepared for that scenario, you want to reduce the damage as much as possible and prevent the attackers from using the
cluster for further attacks on external services and legacy systems outside of the cluster. For that you need egress
traffic control.

Compliance reasons might require you to control the egress traffic too. For example, the [Payment Card
Industry (PCI) Data Security Standard](https://www.pcisecuritystandards.org/pci_security/) requires that inbound
and outbound traffic must be restricted to that which is necessary:

> _1.2.1 Restrict inbound and outbound traffic to that which is necessary for the cardholder data environment, and specifically deny all other traffic._

In this post, I explore the following aspects of securing egress traffic by Istio:

1. describe attacks that involve egress traffic
1. present requirements for egress traffic control
1. describe the Istio way for controlling egress traffic securely
1. compare it with alternative solutions such as
[Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) and legacy
egress proxies/firewalls
1. address performance overhead involved

Let's start with attacks that involve egress traffic.

## The attacks

Any IT organization must operate under the assumption that it will be attacked one day, if it is not attacked already, and
An IT organization must assume it will be attacked if it hasn't been attacked already, and that
part of its infrastructure could already be compromised or become compromised in the future.
Once attackers are able to penetrate an application in a cluster, they can proceed to attack external services:
legacy systems, external web services and databases. The attackers may want to steal the data of the application and to
transfer it to their external servers. Attackers' malware may require access to attackers' servers to download
updates. The attackers may use pods in the cluster to perform DDOS attacks or to break into external systems.
Even though you [cannot know](https://en.wikipedia.org/wiki/There_are_known_knowns) all the possible types of
attacks, you want to reduce possibilities for any attacks, both for known and unknown ones.

The attackers can be external, that is they can gain access to the application’s container from the outside, through a
bug in the application. The attackers can also be internal, that is, for example, malicious DevOps people inside the
organization.

To prevent the attacks described above, some form of egress traffic control must be applied. Let me present egress
traffic control in the following section.

## The solution: egress traffic control

To prevent attacks involving egress traffic, you must monitor all the egress traffic and enforce all security policies.
Once you monitor the egress traffic, you can  analyze it, possibly offline, and detect the attacks, even if
you were unable to prevent them in real time.
Another good practice to reduce possibilities of attacks is to specify policies that limit access following the
[Need to know](https://en.wikipedia.org/wiki/Need_to_know#In_computer_technology]) principle: only the applications that
need external services should be allowed to access the external services they need.

Let me now turn to the requirements for egress traffic control we collected.

## Requirements for egress traffic control

My colleagues at IBM and I collected requirements for egress traffic control from several customers, and combined them with
[egress traffic control requirements from Kubernetes Network Special Interest Group](https://docs.google.com/document/d/1-Cq_Y-yuyNklvdnaZF9Qngl3xe0NnArT7Xt_Wno9beg).
All the requirements are satisfied by Istio 1.1.

1. Support for [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) with [SNI](https://en.wikipedia.org/wiki/Server_Name_Indication) or for [TLS origination](/help/glossary/#tls-origination) by Istio.
1. **Monitor** SNI and the source workload of every egress access
1. Define and enforce **policies per cluster**, e.g.:

   * all applications in the cluster may access `service1.foo.com` (a specific host)
   * all applications in the cluster may access any host of the form `*.bar.com` (a wildcarded domain)

     All unspecified access must be blocked.

1. Define and enforce **policies per source**, _Kubernetes-aware_:

   * application `A` may access `*.foo.com`.
   * application `B` may access `*.bar.com`.

    All other access must be blocked, in particular access of application `A` to `service1.bar.com`.

1. **Prevent tampering**. In case an application pod is compromised, prevent the compromised pod from escaping monitoring,
from sending fake information to the monitoring system, and from breaking the egress policies.
1. Preferably: perform traffic control **transparently** to the applications

Let me explain each of the requirements. The first requirement states that only TLS traffic to the external services must be
supported. The requirement is based on the observation that all the traffic that leaves the cluster usually must be
encrypted.
This means that either the applications will perform TLS origination or Istio must perform TLS origination
for them. Note that in the case an application performs TLS origination, the Istio proxies cannot see the original traffic,
only the encrypted one, so the proxies see the TLS protocol only. For the proxies it does not matter if the original
protocol is HTTP or MongoDB, all the Istio proxies can see is TLS traffic.

The second requirement states that SNI and the source of the traffic must be monitored. Monitoring is the first step to
prevent attacks. Even if attackers would be able to access external services from the cluster, if the access is
monitored, there is a chance to discover the suspicious traffic and take a corrective action.

Note that in the case of TLS originated by an application, the Istio sidecar proxies can only see TCP traffic and a
TLS handshake that includes SNI.
The source of the traffic could be identified by a label of the source pod, by a service account of the pod or by some
other source identifier. We call this property of an egress control system _being Kubernetes-aware_: the system must
understand Kubernetes artifacts like pods and service accounts. If the system is not Kubernetes-aware, it can only monitor
the IP address as the identifier of the source.

The third requirement states that Istio operators must be able to define policies for egress traffic per whole cluster. The
policies state which external services may be accessed by any pod in the cluster. The external services can be
identified either by a [Fully qualified domain name](https://en.wikipedia.org/wiki/Fully_qualified_domain_name) of the
service, e.g. `www.ibm.com` or by a wildcarded domain, e.g. `*.ibm.com`. Only the external services specified may be
accessed, all other egress traffic is blocked.

The rationale for this requirement is that you want to prevent
attackers from accessing malicious sites, for example for downloading updates/instructions for their malware. You also
want to limit the number of external sites that the attackers can access and attack.
You want to allow access only to the external services that the applications in the cluster need to
access and to block access to all the other services, this way you reduce the
[attack surface](https://en.wikipedia.org/wiki/Attack_surface). While the external services
can have their own security mechanisms, you want to exercise [Defense in depth](https://en.wikipedia.org/wiki/Defense_in_depth_(computing)) and to have multiple security layers: a security layer in your cluster in addition to
the security layers in the external systems.

Note that according to the requirement the external services must be identified by domain names. We call this property
of an egress control system _being DNS-aware_.
If the system is not DNS-aware, the external services must be specified by IP addresses.
Using IP addresses is not convenient and often is not feasible, since IP addresses of a service can change. Sometimes
all the IP addresses of a service are not even known, for example in the case of
[CDNs](https://en.wikipedia.org/wiki/Content_delivery_network).

The fourth requirement extends the third requirement, by adding the source of the egress traffic to the policies: the policies
can specify which source can access which external service. The source must be identified as in the second requirement, for
example, by a label of the source pod or by service account of the pod. It means that policy enforcement must also be
_Kubernetes-aware_. If policy enforcement is not Kubernetes-aware, the policies must identify the source of traffic by
the IP of the pod, which is not convenient, especially since the pods can come and go and their IPs are not static.

The fifth requirement states that even if the cluster is compromised and the attackers controls some of the pods, they
must not be able to cheat the monitoring or to violate policies of the egress control system. We say that such a
system provides _secure_ egress traffic control.

The sixth requirement states that the traffic control should be provided without changing the application containers, in
particular without changing the code of the applications and without changing the environment of the containers.
We call such an egress traffic control system _transparent_.

In this blog post I show that Istio can function as an example of an egress traffic control system that satisfies all
of these requirements, in particular it is transparent, DNS-aware, and Kubernetes-aware.

## Legacy solutions for egress traffic control

The most natural solution for egress traffic control is
[Kubernetes Network Policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/). Using
Kubernetes Network Policies, cluster operators can specify which external services can be accessed by which pods. The
pods may be identified by pod labels, namespace labels, or by IP ranges. The external services can be specified by IP
ranges: Kubernetes Network Policies are not DNS-aware. The first requirement is satisfied since any TCP traffic can be
controlled by Kubernetes Network Policies. The requirements 3 and 4 are satisfied partially: the policies can be
specified per cluster or per pod, however the external services cannot be identified by domain names.
The fifth requirement is satisfied if the attackers are not able to break from a malicious container into the Kubernetes
node and to interfere with the implementation of the Kubernetes Network Policies inside the node.
The sixth requirement is satisfied as well: there is no need to change the code or the
container environment. We can say that Kubernetes Network Policies provide transparent, Kubernetes-aware egress traffic
control, which is not DNS-aware.

Another approach that predates Kubernetes network policies is a **DNS-aware egress proxy/firewall**. In this
approach, applications are configured to direct the traffic to the proxy and to use some proxy protocol, e.g.
[SOCKS](https://en.wikipedia.org/wiki/SOCKS).
Since the applications must be configured, this solution is not transparent. Moreover, egress proxies are not
Kubernetes-aware, since neither pod labels nor pod service account are known to the egress proxy. Such egress proxies
cannot fulfill the fourth requirement, that is they cannot enforce policies by source if the source is specified by a
Kubernetes artifact. The egress proxies can fulfill the first, second, third and fifth requirements,
but not the fourth and the six requirements.
They are DNS-aware, but not transparent and not Kubernetes-aware.

## Egress traffic control by Istio

To implement secure egress traffic control in Istio, you must
[direct TLS traffic to external services through an egress gateway](/docs/examples/advanced-gateways/egress-gateway/#egress-gateway-for-https-traffic).
(To support wildcard domains, you must create
[a custom version of an egress gateway](/docs/examples/advanced-gateways/wildcard-egress-hosts/)). Alternatively, you
can [direct HTTP traffic through an egress gateway](/docs/examples/advanced-gateways/egress-gateway/#egress-gateway-for-http-traffic)
and [let the egress gateway perform TLS origination](/docs/examples/advanced-gateways/egress-gateway-tls-origination/#perform-tls-origination-with-an-egress-gateway).

In all cases you have to apply some
[additional security mechanisms](/docs/examples/advanced-gateways/egress-gateway/#additional-security-considerations),
like Kubernetes Network Policies or an L3 firewall that will enforce that traffic from the cluster to the outside is
allowed for the egress gateway only. See
[an example for Kubernetes Network Policies configuration](/docs/examples/advanced-gateways/egress-gateway/#apply-kubernetes-network-policies).

You must also increase the security measures applied to the Istio control plane and the
egress gateway by running them on nodes separate from the application nodes, in a separate namespace, monitoring them
more thoroughly, etc. After all, if the attackers are able to attack Istio Mixer or the egress gateway, they could
violate any policy.

Once you direct egress traffic through an egress gateway and apply the additional security mechanisms,
you can securely monitor the traffic and define security policies on it.
If the application sends HTTP requests and the egress gateway performs TLS origination, you can monitor HTTP
information, e.g HTTP methods, headers and URL paths, and you can
[define policies](/blog/2018/egress-monitoring-access-control) based on the HTTP information. If the application
performs TLS origination, for TLS traffic you can
[monitor SNI and the service account](/docs/examples/advanced-gateways/egress_sni_monitoring_and_policies/) of the
source pod, and define policies based on them.

The following diagram shows Istio's security architecture, augmented with L3 firewall (part of the
[additional security mechanisms](/docs/examples/advanced-gateways/egress-gateway/#additional-security-considerations)
provided outside of Istio by the cluster/cloud provider).
The L3 firewall can have a trivial configuration that would allow incoming traffic into Istio ingress gateway pods only and
outgoing traffic from Istio egress gateway pods only. Note that the Istio proxy of the egress gateway performs
policy enforcement and reporting in the same way as the sidecar proxies in the application pods.

{{< image width="80%" link="./SecurityArchitectureWithL3Firewalls.svg" caption="Istio Security Architecture with Egress Gateway and L3 Firewall" >}}

Now let's examine possible attacks and let me show you how secure egress control in Istio prevents them.

### Possible attacks and their prevention

Consider the following security policy with regard to egress traffic:

1. Application A is allowed to access `*.ibm.com` (all the external services with URL matching `*.ibm.com`,
  e.g. `www.ibm.com`)
1. Application B is allowed to access `mongo1.composedb.com`
1. All egress traffic must be monitored

Now consider a scenario in which one of application A's pods is compromised. Suppose the attackers have the
following goals:

1. Application A will try to access `*.ibm.com` unmonitored
1. Application A will try to access `mongo1.composedb.com`

Note that application A is allowed to access `*.ibm.com`, so the attacker is able to access it. There is no way
to prevent such access since there is no way to distinguish, at least initially, between the original and the
compromised versions of application A. However, you want to monitor any access to external services to be able to
detect suspicious traffic, for example by applying anomaly detection tools on logs of the egress traffic.
The attackers, on the contrary, want to access external services unmonitored, so the attack will not be detected.
The second goal of the attackers is to access `mongo1.composedb.com`, which is forbidden for application A. Istio
must enforce the policy that forbids access of application A to `mongo1.composedb.com` and must prevent the attack.

Now let's see which attacks the attackers will try to perform to achieve their goals and how Istio secure egress traffic
control will prevent each kind of attack. The attackers may try to:

1. **Bypass** the container's sidecar proxy and access external services directly. This attack is prevented by a
   Kubernetes Network Policy or by an L3 firewall that allow egress traffic to exit the mesh only from the egress
   gateway.
1. **Compromise** the egress gateway and force it to send fake information to the monitoring system or to disable
   enforcement of the security policies.
   This attack is prevented by applying the special security measures to the egress gateway pod.
1. Since the previous attacks are prevented, the attackers have no other option but to direct the traffic through the
   egress gateway. The traffic will be monitored by the egress gateway, so the goal of the attackers to access
   external services unmonitored cannot be achieved. The attackers may want to try to achieve their second goal, that is
   to access `mongo1.composedb.com`. To achieve it, they may try to **impersonate** as application B since
   application B is allowed to access `mongo1.composedb.com`. This attack, fortunately, is prevented by Istio's [strong
   identity support](/docs/concepts/security/#istio-identity).

After I showed how Istio egress traffic control can prevent possible attacks, let me describe its advantages over
Kubernetes Network Policies and legacy egress proxies/firewalls.

### Advantages of Istio egress traffic control

Istio egress traffic control is **DNS-aware**: you can define policies based on URLs or on wildcard domains like
`*.ibm.com`. In this sense, it is better than Kubernetes Network Policies which are not DNS-aware.

Istio egress traffic control is **transparent** with regard to TLS traffic, since Istio is transparent:
you do not need to change the applications or to configure their containers.
For HTTP traffic with TLS origination, DevOps people must configure the applications to use HTTP instead of HTTPS
when the applications run with Istio sidecars injected.

Istio egress traffic control is **Kubernetes-aware**: the identity of the source of egress traffic is based on
Kubernetes service accounts. Istio egress traffic control is better than the legacy DNS-aware proxies/firewalls which
are not transparent and not Kubernetes-aware.

Istio egress traffic control is **secure**: it is based on the strong identity of Istio and, when the cluster operators
provide
[additional security measures](/docs/examples/advanced-gateways/egress-gateway/#additional-security-considerations),
it is tamper-proof.

On top of these beneficial features, Istio egress traffic control provides additional advantages:

*  It allows defining access policies in the same language for ingress/egress/in-cluster traffic. The cluster operators
   need to learn a single policy and configuration language for all types of traffic.
*  Istio egress traffic control is integrated with Istio policy and telemetry adapters and can work out-of-the-box.
*  When you use external monitoring/access control systems with Istio, you must write the adapters for them only once,
   and then apply the adapters for all types of traffic: ingress/egress/in-cluster.
*  Istio operators can apply Istio traffic management features to egress traffic, such as
   load balancing, passive and active health checking, circuit breaker, timeouts, retries, fault injection, and others.

We call a system that has the advantages above **Istio-aware**.

Let me summarize the features of Istio egress traffic control and of the alternative solutions in the following table:

| | Kubernetes Network Policies | Legacy Egress Proxy/Firewall | Istio Egress Traffic Control |
| --- | --- | --- | ---|
| DNS-aware | {{< cancel_icon >}} | {{< checkmark_icon >}} | {{< checkmark_icon >}} |
| Kubernetes-aware | {{< checkmark_icon >}} | {{< cancel_icon >}} | {{< checkmark_icon >}} |
| Transparent | {{< checkmark_icon >}} | {{< cancel_icon >}} | {{< checkmark_icon >}} |
| Istio-aware | {{< cancel_icon >}} | {{< cancel_icon >}} | {{< checkmark_icon >}} |

### Performance considerations

Note that Istio egress control has its price, which is the increased latency of the calls to external services and
increase of CPU usage by the cluster pods.
After all, the traffic has to pass through two proxies, namely the sidecar proxy of the
application and the proxy of the egress gateway. In the case of
[TLS egress traffic to wildcard domains](/docs/examples/advanced-gateways/wildcard-egress-hosts/),
you have to add
[an additional proxy](/docs/examples/advanced-gateways/wildcard-egress-hosts/#wildcard-configuration-for-arbitrary-domains),
making the count of proxies between the application and the external service three. The traffic between the second and
third proxies is on the local network of the pod, so it should not have significant impact on the latency.

See a [performance evaluation](/blog/2019/egress-performance/) of different configurations of Istio egress
traffic control. I would encourage you to measure carefully different configurations for your own applications and your
external services, and decide whether you can afford the performance overhead for your use cases. You should weigh the
required level of security versus your performance requirements, and also compare with the performance overhead of
alternative solutions.

Let me provide our take on the performance overhead of Istio egress traffic control.
The latency of access to external services could be already high, so adding the overhead
of two or three proxies inside the cluster could be not very significant.
After all, in the microservice architecture you can have chains of dozens of calls between microservices, so adding an
additional hop with two proxies, the egress gateway, should not have a large impact.

Moreover, we are working to reduce
performance overhead of Istio, so I hope the overhead of egress traffic control in Istio will be reduced in the future.
Possible optimizations are to extend Envoy to handle wildcard domains so there will be no need for the
third proxy; or to use mutual TLS for authentication only without encrypting the TLS traffic (since it is already
encrypted).

## Summary

I hope that you are convinced that controlling egress traffic is very important for the security of your cluster.
I also hope that I managed to convince you that Istio can serve as an effective tool for controlling egress traffic
securely, and that Istio has multiple advantages over the alternative solutions.
In my opinion, you can even choose secure egress traffic control as the first use case for applying Istio to your
cluster.
Istio will already be beneficial for you, even before you start using all other features, such as
traffic management, security, policies and telemetry, applied to traffic between microservices inside the cluster.
You should pay attention, however, to performance considerations of Istio egress traffic control and measure performance
overhead for your use cases.