---
layout: blog
title: "Kubernetes Networking Architecture"
slug: kubernetes-networking-architecture
---
**Author**: Susie Su
# Cloud Computing: Kubernetes Networking Architecture
From Docker to Kubernetes, container technologies are evolving. Most developers are familiar with Docker but don't have the whole perspective of Kubernetes. And the most difficult thing to understand about Kubernetes is its network architecture. Here, with my 10 years of DevOps work experience and double CCIE in the field of networking and cybersecurity, I will provide a detailed explanation of the comprehensive architecture of Kubernetes Networking, ensuring all aspects are clearly understood. 

[<u>Kubernetes Networking
Architecture</u>](#kubernetes-networking-architecture)

> [<u>Network Namespace (netns)</u>](#network-namespace-netns)
>
> [<u>Veth (virtual ethernet) pairs</u>](#veth-virtual-ethernet-pairs)
>
> [<u>Types of Virtual Network Interfaces in
> Linux</u>](#types-of-virtual-network-interfaces-in-linux)
>
> [<u>Network Bridge Interface</u>](#network-bridge-interface)
>
> [<u>A network bridge is similar to a layer-2
> switch</u>](#a-network-bridge-is-similar-to-a-layer-2-switch)
>
> [<u>Container Runtime</u>](#container-runtime)
>
> [<u>Container Network Mode</u>](#container-network-mode)
>
> [<u>Docker Bridge Mode</u>](#docker-bridge-mode)
>
> [<u>Kubernetes CNI</u>](#kubernetes-cni)
>
> [<u>Common CNI Plugins</u>](#common-cni-plugins)
>
> [<u>Netfilter and</u> <u>iptables</u>](#netfilter-and-iptables)
>
> [<u>Routing</u>](#routing)
>
> [<u>Pod-to-Pod Across Nodes
> Routing</u>](#pod-to-pod-across-nodes-routing)
>
> [<u>Overlay Networks</u>](#overlay-networks)
>
> [<u>Linux Overlay Network
> technology</u>](#linux-overlay-network-technology)
>
> [<u>Node-to-Node Routing</u>](#node-to-node-routing)
>
> [<u>Service Routing</u>](#service-routing)
>
> [<u>Network Policies</u>](#network-policies)
>
> [<u>Ingress Controller</u>](#ingress-controller)
>
> [<u>External Routing</u>](#external-routing)

[<u>Classic Networking Scenarios in
Kubernetes</u>](#classic-networking-scenarios-in-kubernetes)

> [<u>Between Container and
> Container</u>](#between-container-and-container)
>
> [<u>Between Pod and Pod</u>](#between-pod-and-pod)
>
> [<u>Pods in the same node</u>](#pods-in-the-same-node)
>
> [<u>Pods across the nodes</u>](#pods-across-the-nodes)
>
> [<u>Between Pod and Service</u>](#between-pod-and-service)
>
> [<u>Pod to Service</u>](#pod-to-service)
>
> [<u>Service to Pod</u>](#service-to-pod)
>
> [<u>Between Internet and Service</u>](#between-internet-and-service)

[<u>Conclusion</u>](#conclusion)


# Kubernetes Networking Architecture

Kubernetes runs on the Linux system. There are five key networking
concepts in Linux, which will help you understand how the Kubernetes
network was designed to be compatible with these key components and meet
the final requirements.

## Network Namespace (netns)

|                   |                                |
|-------------------|--------------------------------|
| Linux Network     | Kubernetes Network             |
| Network Namespace | Node in Root Network Namespace |
|                   | Pod in Pod Network Namespace   |

A network namespace is a feature in Linux that allows you to create
isolated network environments within a single Linux system. Each network
namespace has its own network stack, including network interfaces,
routing tables, firewall rules, and other network-related resources.
This isolation allows you to run multiple independent network
environments on the same physical or virtual machine, keeping them
separate from each other.

Tools like Docker and Kubernetes use network namespaces to provide
isolated networking environments for each container. This isolation
ensures that containers cannot interfere with each other's network
configurations.

**Network Namespace Experiment**

How do we create a network namespace and achieve communication between
different network namespaces?

Step 1: Create two network namespaces:

```shell
ip netns add netns1  
ip netns add netns2
```

{{< figure src="image5.svg" title="create-netns" >}}


Step2: Create a veth pair:

```shell
ip link add veth1 type veth peer name veth2
```

1.  Assign veth 1 to the network namespace netns1 and veth 2 to the
    network namespace netns2:

```shell
ip link set veth1 netns netns1  
ip link set veth2 netns netns2
```

2.  Assign IP address to the interfaces within the namespaces:

```shell
ip netns exec netns1 ip addr add 10.1.1.1/24 dev veth1  
ip netns exec netns2 ip addr add 10.1.1.2/24 dev veth2
```

But veth1 cannot successfully ping veth2, as shown in the screenshot
below.

{{< figure src="image6.svg" title="netns1-ping-netns2" >}}


```shell
ip netns exec netns2 ip link show veth2 # Check the status of veth2, which is down
```

{{< figure src="image7.svg" title="netns-ip-link" >}}

3.  Activate the network interface veth1 within netns1 network namespace
    and veth2 within netns2.

```shell
ip netns exec netns1 ip link set dev veth1 up  
ip netns exec netns2 ip link set dev veth2 up
```

4.  Try to ping veth2 from netns1:

```shell
​​ip netns exec netns1 ping 10.1.1.2
```

{{< figure src="image8.svg" title="netns1-ping-netns2-success" >}}

So far, we've successfully completed the experiment; let's continue
further.

**Brainstorming**

As we've learned from the above experiment, namespaces isolate the
network, and we've used a veth pair to connect two virtual hosts. Now,
if we configure the IP addresses with different subnets, what will
happen?

{{< figure src="image9.svg" title="ping-unsuccessful" >}}

Without a router or subnet reconfiguration, the two hosts in different
subnets will not be able to communicate over a simple network cable
connection.

|                                                                                                                                                                                                                                                                                                |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| It reminds me of a loopback address. For example, the loopback interface is a special interface for same-host communication. Packets sent to the loopback interface will not leave the host, and processes listening on 127.0.0.1 will be accessible only to other processes on the same host. |

## Veth (virtual ethernet) pairs

|               |                    |
|---------------|--------------------|
| Linux Network | Kubernetes Network |
| Veth Pair     | Veth Pair(on Pods) |

Veth pairs facilitate communication between different network namespaces
or containers. Traffic sent into one end of the veth pair can be
received at the other end, allowing data to flow between isolated
environments. Otherwise, it cannot interconnect between different
network namespaces by default.

{{< figure src="image10.svg" title="simple-veth-pair" >}}

In simple words, the veth pair seems like a virtual network cable that
connects virtual hosts in two different namespaces.

|                                                                                                                                                         |
|---------------------------------------------------------------------------------------------------------------------------------------------------------|
| Packets transmitted from one device in the pair are immediately received on the other host. When either host is down, the link state of the pair is down. |

Linux itself features the concept of network interfaces, which can be
either physical or virtual. For instance, when you use the 'ifconfig'
command, you can view a list of all network interfaces and their
configurations, including IP addresses. Veth is one such virtual network
interface in Linux.

### Types of Virtual Network Interfaces in Linux

**Virtual Ethernet Interfaces (veth)**

These are often used in containerization technologies like Docker. They
create a pair of virtual Ethernet interfaces, and packets sent to one
end of the pair are received on the other end.

```shell
ip link add veth0 type veth peer name veth1  
ip link set dev veth0 up  
ip link set dev veth1 up
```

**Loopback Interface (lo)**

The loopback interface is used for local network communication, allowing
a system to communicate with itself. It is typically assigned the IP
address 127.0.0.1.

```shell
ifconfig lo
```

{{< figure src="image11.svg" title="ifconfig-lo" >}}

```shell
ip addr show lo
```

{{< figure src="image12.svg" title="ip-addr-show-lo" >}}

**Tunnel Interfaces (TUN, tap)**

Tunnel interfaces are used to create point-to-point or
point-to-multipoint network tunnels, such as VPNs. tun interfaces are
used for routing, while tap interfaces are used for Ethernet bridging.

To create a TUN interface for a simple tunnel:

> ip tuntap add mode tun tun0  
> ip link set dev tun0 up

To create a TAP interface for an Ethernet bridge:

> ip tuntap add mode tap tap0  
> ip link set dev tap0 up

**Virtual LAN Interfaces (vlan)**

VLAN interfaces are used to partition a physical network into multiple
virtual LANs. They allow multiple VLANs to exist on a single physical
network interface.

> ip link add link eth0 name eth0.10 type vlan id 10  
> ip link set dev eth0.10 up

**WireGuard Interfaces (wg)**

WireGuard is a modern VPN protocol, and wg interfaces are used to
configure and manage WireGuard tunnels.

Create a WireGuard interface:

> ip link add dev wg0 type wireguard

Generate private and public keys

> wg genkey \| sudo tee /etc/wireguard/privatekey \| wg pubkey \| sudo
> tee /etc/wireguard/publickey

Configure the WireGuard interface

> ip address add 10.0.0.1/24 dev wg0  
> wg set wg0 private-key /etc/wireguard/privatekey  
> wg set wg0 listen-port 51820

**Dummy Interfaces (dummy)**

Dummy interfaces are used for various purposes, such as simulating
network interfaces or providing a placeholder for network configuration.

> modprobe dummy  
> ip link add dummy0 type dummy  
> ip link set dev dummy0 up

**Bonded Interfaces (bond)**

Bonding interfaces are used for network link aggregation and fault
tolerance. They combine multiple physical network interfaces into a
single logical interface.

> modprobe bonding  
> ip link add bond0 type bond  
> ip link set dev eth0 master bond0  
> ip link set dev eth1 master bond0

**Virtual TTY Interfaces (PTY – ‘Pseudo-Terminal’)**

These are virtual terminal interfaces used for terminal emulation, often
in the context of SSH or terminal multiplexers like tmux.

{{< figure src="image13.svg" title="tty" >}}
{{< figure src="image14.svg" title="tmux" >}}

**Bridge Interfaces (br)**

Bridge interfaces are used for bridging network traffic between two or
more physical or virtual network interfaces. They are created when
setting up software bridges using tools like brctl or ip.

> ip link add name br0 type bridge  
> ip link set dev eth0 master br0 \# eth0 is the physical network
> interface, and it can also configure veth (virtual network interface)
> instead, for example, with the command 'ip link set veth master
> br0'.  
> ip link set dev eth1 master br0  
> ip link set dev br0 up

After becoming familiar with the various virtual network interfaces
above, let's shift our focus to the Linux Network Bridge, which can be
managed and created using 'brctl.' In Kubernetes networking, we use CNI
to manage Linux network interfaces for container networking, as
illustrated in the following table.

<table>
<colgroup>
<col style="width: 26%" />
<col style="width: 38%" />
<col style="width: 35%" />
</colgroup>
<tbody>
<tr class="odd">
<td>Linux Network</td>
<td colspan="2">Kubernetes Network</td>
</tr>
<tr class="even">
<td rowspan="4">Network Interface(Physical+Virtual)</td>
<td rowspan="4"><p>CNI plugins</p>
<p>(Container Network Interface)</p></td>
<td>Cilium</td>
</tr>
<tr class="odd">
<td>Flannel</td>
</tr>
<tr class="even">
<td>Calico</td>
</tr>
<tr class="odd">
<td>Weave Net</td>
</tr>
</tbody>
</table>

|                                                                                                                                            |
|--------------------------------------------------------------------------------------------------------------------------------------------|
| Linux network interfaces are the underlying foundation, while CNI provides a standardized way to configure container network connectivity. |

## Network Bridge Interface

A veth pair is primarily used for connecting two network namespaces or
containers and provides network isolation between them. A bridge, on the
other hand, is used to connect **multiple network interfaces** and
create a single network segment without inherent network isolation,
although additional configurations can be applied for segmentation.
Bridges and veth pairs can be used together to create more complex
network setups, such as connecting multiple containers within a single
bridge.

|                  |                |                          |
|------------------|----------------|--------------------------|
| Linux Network    | Docker Network | Kubernetes Network       |
| Bridge Interface | docker 0       | cni0 (on Flannel plugin) |

### A network bridge is similar to a layer-2 switch

For example, the "docker0" bridge is a default bridge network created by
Docker when you install Docker on a Linux system. It is a virtual
Ethernet bridge that enables communication between Docker containers and
between containers and the host system.

{{< figure src="image15.svg" title="dokcer-bridge-networking-in-linux" >}}

When discussing bridge mode in both Docker and Kubernetes, it's
essential to understand the concept of container runtime because the
container runtime is responsible for the actual low-level management of
containers, including networking.

### Container Runtime

Kubernetes is the most widely used container orchestration engine. It
has an entire foundation and ecosystem that surrounds it with additional
functionality to extend its capabilities through its defined APIs. One
of these APIs is the Container Runtime Interface (CRI), which defines
what Kubernetes wants to do with a container and how.

Several common container runtimes with Kubernetes

- **Docker**: Historically, Docker was the first container runtime that
  Kubernetes supported directly. However, Kubernetes uses a Container
  Runtime Interface (CRI) to interact with container runtimes, which
  Docker did not originally support. To integrate Docker with
  Kubernetes, a component called Dockershim was used. Dockershim acted
  as an adapter layer that allowed Kubernetes to communicate with
  Docker's daemon using the CRI.

> In more recent developments, Kubernetes announced the deprecation of
> Dockershim as an intermediary. The primary reason for this was to
> streamline the Kubernetes architecture and use container runtimes that
> directly implement the CRI. However, Docker containers and images
> remain fully compatible with Kubernetes because Docker produces
> OCI-compliant containers. This means that the containers built with
> Docker can be run by other CRI-compatible runtimes like containerd and
> CRI-O, which Kubernetes supports natively.

- **containerd**: An industry-standard container runtime focused on
  simplicity and robustness, containerd is used by Docker and supports
  CRI, making it compatible with Kubernetes.

- **CRI-O**: A lightweight container runtime specifically designed for
  Kubernetes, fully conforming to the Kubernetes CRI, which allows
  Kubernetes to use any OCI (Open Container Initiative) compliant
  runtime.

- **Mirantis Container Runtime**(formerly Docker Engine — Enterprise):
  It's the enterprise-supported version of Docker runtimes and can
  integrate with Kubernetes via the CRI.

The vast majority of solutions (not just Kubernetes) rely on containerd,
including developer-friendly tooling like Docker Desktop. Exceptions
include Red Hat OpenShift, which uses CRI-O, and Podman, which directly
interacts with low-level container runtimes like runC. Ali Cloud has
replaced the docker with containerd for the latest ACK.

### Container Network Mode

The industry, for now, has settled on Docker as the container runtime
standard. Thus, we'll dive into the Docker networking model, and explain
how the CNI(cni0) differs from the docker network model(docker0).

Docker container supports various network modes:

- **Bridge Mode** (*default*): Containers are attached to a bridge
  network, providing network isolation and default mode for new
  containers.

- **Host Mode** (*--net=host*): Containers share the host's network,
  offering maximum performance but no network isolation.

- **Container Mode** (*--net=container*): Containers share the network
  stack of another container, useful for direct communication.

- **None Mode** (*--net=none*): Containers have no network access,
  isolating them from the host and others.

Docker containers also offer advanced network modes for specific use
cases:

- **Macvlan**: allows Docker containers to have direct, unique, and
  routable IP addresses on the same physical network as the host.

- **Overlay**: allows for the extension of the same network across hosts
  in a container cluster. The overlay network virtually sits on top of
  the underlay/physical networks.

Let’s focus on docker bridge mode first.

#### Docker Bridge Mode

In bridge mode, Docker creates a bridge network interface (commonly
named "Docker0") and assigns a subnet within a private IP address space
(often following the RFC1918 model) to this bridge. For each Docker
container, a pair of virtual Ethernet (veth) devices is generated.

One end of the veth pair is connected to the Docker0 bridge, and the
other end is mapped to the container's eth0 network interface using
network namespace technology. Finally, an IP address is allocated to the
eth0 interface of the container within the address range of the Docker0
bridge, allowing the container to communicate with other containers and
the host system.

So far, we have solved the problem of a single-node container network
through veth pair + bridge, which is also the principle of a Docker
network. However, docker0 on one host has nothing to do with docker0 on
other hosts. For Kubernetes, while it uses Docker as a container
runtime, it handles networking differently. Thus, let’s further discuss
a new term of Kubernetes - CNI(Container Network Interface).

#### Kubernetes CNI 

Kubernetes imposes its own network model on top of Docker's, which
ensures that every pod gets a unique IP address. This model is typically
implemented with the help of Container Network Interface (CNI) plugins.
Docker's network modes are still relevant for local container
management, but when it comes to Kubernetes clusters, the networking is
largely managed by Kubernetes services and CNI-compliant plugins.

##### Common CNI Plugins

**Flannel**: Provides a simple and easy way to configure a layer 3
network fabric designed for Kubernetes.

**Calico**: Offers networking and network policy, typically for larger
or more dynamic networks with advanced security requirements.

**Weave Net**: Creates a network bridge on each node and connects each
container to the bridge via a virtual network interface, allowing for
more complex networking scenarios, including network partitioning and
secure cross-host communication.

**Cilium**: Uses BPF (Berkeley Packet Filter) to provide highly scalable
networking and security policies.

**AWS VPC CNI, Azure CNI, GCP, and Ali Cloud CNI**:
Cloud-provider-specific plugins that integrate Kubernetes clusters with
the network services of their respective cloud platforms, enabling
seamless operation and management of network resources.

## Netfilter and iptables

|                     |                                                          |
|---------------------|----------------------------------------------------------|
| Linux Network       | Kubernetes Network                                       |
| Netfilter(Security) | iptables                                                 |
| Netfilter/iptables  | iptables/IPVS mode (Loadbalancer created by kube-proxy ) |

**Netfilter** is the underlying framework in the Linux kernel
responsible for packet filtering, NAT, and connection tracking, while
**iptables** is a user-space command-line tool that leverages the
Netfilter framework to define and manage firewall rules. It is the
underlying technology used by iptables to perform these functions:

- Connection Tracking: Netfilter maintains a connection tracking table
  that keeps track of the state of network connections. This is
  essential for implementing NAT and ensuring that packets are correctly
  routed to the appropriate destination.

- NAT (Network Address Translation): Netfilter allows for NAT
  operations, which are often used for masquerading outgoing traffic
  from pods to appear as if it's coming from the node's IP address when
  communicating with external networks.

- Load Balancing: In Kubernetes, Netfilter also plays a role in load
  balancing traffic to different pods within a Service.

{{< figure src="image16.svg" title="netfilter-iptables" >}}

*Netfilter vs iptables in Linux*

In Kubernetes, there are several components and concepts that serve
similar roles and functions as Netfilter/iptables in managing network
traffic. These include:

**iptables**: In Kubernetes, it is used to implement various network
policies and rules for managing traffic between pods and between pods
and external networks.

iptables running over Netfilter enable Kubernetes to enforce network
policies, perform load balancing for Services, and handle NAT
operations. They are critical components of Kubernetes networking,
ensuring that traffic flows correctly within the cluster and between
pods and external networks.

**kube-proxy**: kube-proxy is a Kubernetes component responsible for
managing network rules and routing traffic to services and pods within
the cluster. It uses iptables to set up the necessary rules for traffic
routing.

**CNI (Container Network Interface) plugins**: CNI plugins, such as
Flannel, Calico, Weave Net, and others, provide networking and network
policy functionalities within Kubernetes. They often interact with the
underlying networking components, including iptables and the Linux
kernel's network stack.

**Network policies**: Kubernetes Network Policies allow you to define
rules that control the traffic flow to and from pods. These policies are
enforced using iptables rules on the nodes. Network Policies provide
fine-grained control over which pods can communicate with each other.

<table>
<colgroup>
<col style="width: 24%" />
<col style="width: 24%" />
<col style="width: 25%" />
<col style="width: 25%" />
</colgroup>
<tbody>
<tr class="odd">
<td><strong>Name</strong></td>
<td><strong>NetworkPolicy Support</strong></td>
<td><strong>Data storage</strong></td>
<td><strong>Network setup</strong></td>
</tr>
<tr class="even">
<td>Cilium</td>
<td>Yes</td>
<td>etcd or consul</td>
<td>IPvlan(beta), veth, L7-aware</td>
</tr>
<tr class="odd">
<td>Flannel</td>
<td>No</td>
<td>etcd</td>
<td>Layer3 IPv4 Overlay network</td>
</tr>
<tr class="even">
<td>Calico</td>
<td>Yes</td>
<td>etcd or Kubernetes<br />
API</td>
<td>Layer3 network using BGP</td>
</tr>
<tr class="odd">
<td>Weave Net</td>
<td>Yes</td>
<td>No external<br />
cluster store</td>
<td>Mesh Overlay network</td>
</tr>
</tbody>
</table>

> *NetworkPolicy Support Status in CNI Plugins*

To use Network Policy, Kubernetes introduces a new resource object
called "NetworkPolicy," which allows users to define policies for
network access between pods. However, merely defining a network policy
is insufficient to achieve actual network isolation. It also requires a
policy controller (PolicyController) to implement the policies.

The policy controller is provided by third-party networking components.
Currently, open-source projects such as Calico, Cilium, Kube-router,
Romana, Weave Net, and others support the implementation of network
policies (as shown in the table above). The working principle of Network
Policy is depicted in the diagram below.

{{< figure src="image17.svg" title="network-policy" >}}

*Network Policy Working Principle*

## Routing

Routing in Linux is quite simple. Let's delve directly into Kubernetes
routing. First, you should be aware that in Kubernetes, there are
several IP address blocks

- **Pod Network CIDR**: Assigns IP addresses to pods.

- **Service Cluster IP Range**: Defines IP addresses for services.

- **Node Network CIDR**: Assigns IP addresses to nodes.

- **Service External IP Range**: Specifies external IP addresses for
  services.

In Kubernetes, routing refers to the process of directing network
traffic between pods, nodes, and external networks within the cluster.
Routing plays a crucial role in ensuring that communication between
different components of a Kubernetes cluster functions correctly.

Here are key aspects of routing in Kubernetes:

### Pod-to-Pod Across Nodes Routing

|                                 |                    |                                                                  |
|---------------------------------|--------------------|------------------------------------------------------------------|
| Linux Network                   | Kubernetes Network |                                                                  |
| VXLAN/GRE/IP-in-IP/Open vSwitch | CNI Plugins        | Flannel Network Type(udp/vxlan/host-gw/Cloud Provider VPC/alloc) |
|                                 |                    | Cilium/Calico/Weave Net                                          |

Kubernetes assigns a unique IP address to each pod within a cluster.
Pods can communicate directly with each other using these IP addresses,
regardless of the node they are running on.

The Kubernetes networking model, often implemented using overlay
networks or network plugins, ensures that pod-to-pod traffic is
efficiently routed within the cluster. Let’s elaborate on overlay
networks and their relevant common network plugins.

#### Overlay Networks

Overlay networks provide a means to connect containers across different
hosts. Both Docker Swarm and Kubernetes support overlay networks. In
Docker Swarm, you can create overlay networks to enable container
communication across multiple nodes. In Kubernetes, overlay network
plugins like Flannel or Calico are used for pod-to-pod communication
across nodes. It's worth noting that while Docker container-to-container
communication is similar to pod-to-pod communication in Kubernetes, the
latter provides additional orchestration and management features.

{{< figure src="image18.svg" title="flannel-overlay" >}}

*This is how the overlay network (using flannel plugins) works in
Kubernetes*

- Flannel offers simple overlay networking.

- Calico provides advanced networking with BGP routing and rich network
  policies.

- Weave Net focuses on simplicity and includes DNS-based service
  discovery.

Kubernetes' CNI plugins, such as Flannel, have specific networking
characteristics. The traditional Flannel plugin, when used across nodes,
relies on vxlan (flannel.1) and UDP (flannel0) for communication. This
results in an additional layer of packet encapsulation when crossing
nodes. However, Ali Cloud's Flannel mode, known as 'alloc,' takes a
different approach by directly utilizing the VPC's routing table. This
reduces packet encapsulation and leads to faster and more stable packet
transmission. Cross-node communication is just one aspect of the Flannel
plugin; it also performs functions like assigning IP addresses to pods
and configuring routing.

Actually, the requirement for communication across nodes can be
addressed using Linux's overlay network technologies. While these
technologies may not provide IP assignment to pods like Kubernetes' CNI
plugins, they share many similarities. It's valuable to understand how
Linux originally provided relevant solutions. These Linux L2/L3 routing
technologies are integrated into the Linux kernel and are accessible
without the need for additional software or drivers.

#### Linux Overlay Network technology

VXLAN (Virtual Extensible LAN)

VXLAN is a tunneling technology used for overlay networking, often used
in cloud and virtualized environments. It is supported in the Linux
kernel.

{{< figure src="image19.svg" title="vxlan-tunnel" >}}

GRE (Generic Routing Encapsulation):

GRE is a tunneling protocol that encapsulates a wide variety of network
layer protocols inside virtual point-to-point links.

IP-in-IP:

A simple tunneling protocol that encapsulates an IP packet within
another IP packet.

Open vSwitch (OVS):

A multilayer virtual switch that supports standard management interfaces
and protocols, including NetFlow, sFlow, SPAN, RSPAN, CLI, LACP,
802.1ag, and it can also operate both as a software-based network switch
and as a network overlay for virtual machines.

### Node-to-Node Routing

|               |                                 |
|---------------|---------------------------------|
| Linux Network | Kubernetes Network              |
| L3 Routing    | L3 Routing (underlying network) |

Kubernetes nodes may need to communicate with each other for various
reasons, such as control plane coordination or network traffic routing.

Routing between nodes is typically managed by the underlying network
infrastructure and is necessary for features like LoadBalancer-type
Services, which route traffic to different nodes hosting pods.

### Service Routing

|               |                                                          |
|---------------|----------------------------------------------------------|
| Linux Network | Kubernetes Network                                       |
| Netfilter     | iptables/IPVS mode (Loadbalancer created by kube-proxy ) |

It’s the most challenging to comprehend and the most different from
Docker swarm. Kubernetes Services provide a stable and abstracted way to
access pods. They rely on routing to distribute incoming traffic to the
appropriate pods, regardless of the node they are on.

Services use kube-proxy and iptables rules to perform load balancing and
route traffic to the correct endpoints (pods) based on labels and
selectors.

### Network Policies

It allows you to define rules that control pod-to-pod communication.
These policies act as routing rules enforced by the underlying
networking infrastructure (often implemented using iptables). Network
Policies provide fine-grained control over which pods can communicate
with each other, based on labels and selectors.

### Ingress Controller

It is like Nginx Ingress or Traefik, managing external access to
services within the cluster. They handle routing external traffic based
on rules defined in Ingress resources.

### External Routing

Kubernetes clusters often require communication with external networks,
such as the public internet or on-premises data centers. External
routing is essential for ingress and egress traffic.

External routing is typically managed by the cluster's network
configuration and cloud provider integration.

In summary, routing in Kubernetes is a complex and critical aspect of
cluster networking. It involves directing traffic between pods, nodes,
and external networks to ensure that applications running within the
cluster can communicate effectively and securely. Kubernetes provides
various networking components and abstractions to manage routing and
connectivity within the cluster.

# Classic Networking Scenarios in Kubernetes

In this section, I will cover almost all types of network organization
scenarios based on the theory of Kubernetes.

## Between Container and Container

Let’s see how container 1 communicates with container 2 in the same pod,
as illustrated in the following diagram.
{{< figure src="image20.svg" title="c2c" >}}

Communication between Container1 and Container2

In each pod, every Docker container, and the pod itself share a network
namespace. This means that network configurations such as IP addresses
and ports are identical for both the pod and its individual containers.

This is primarily achieved through a mechanism called the "pause
container," where newly created Docker containers do not create their
own network interfaces or configure their own IP addresses; instead,
they share the same IP address and port range with the pause container.

The pause container is the parent container for all running containers
in the pod. It holds and shares all the namespaces for the pod.

Linux network namespaces provide network isolation at the OS level,
while Kubernetes namespaces offer logical resource isolation within a
Kubernetes cluster, serving different purposes in networking and
resource management.

In summary, the two containers above in the same pod within the same
network namespace are similar to the two processes in Linux, and they
can communicate with each other directly and share the same network
configuration within the pod's network namespace.

## Between Pod and Pod

### Pods in the same node

This is how communication between Pods on the same Node works:

{{< figure src="image21.gif" title="p2p-on-the-same-node" >}}

*Pod1, Pod2, and cni0 are in the same subnet*

### Pods across the nodes

This is how communication between pod1 and pod2 across nodes works:

{{< figure src="image22.gif" title="p2p-across-nodes" >}}
*Pods Communication across nodes*

In the case of communication between Pods on different Nodes, the pod's
network segment and the bridge are within the same network segment,
while the bridge's network segment and the Node's IP are two different
network segments. To achieve communication between different Nodes, a
method needs to be devised to address and communicate based on the
Node's IP.

On the other hand, these dynamically assigned pod IPs are stored as
endpoints for services in the etcd database of the Kubernetes cluster.
These endpoints are crucial for enabling communication between pods.

These are detailed solutions to ensure seamless pod connectivity across
nodes.

1.  Unique Pod IP Allocation:

- Utilize Overlay network technologies such as Flannel plugins.

- Utilize other network plugins

2.  Mapping Pod IPs to Host IPs: Establish a correlation between the
    Pod's IP and the node's IP to facilitate communication among Pods by
    implementing Pod IP mapping to the node IP.

## Between Pod and Service

When we deploy a pod, it generally includes at least two containers: the
pause container and the app container.

The IP address of a pod is not persistent; when the number of pods in
the cluster is reduced, or if a pod or node fails and restarts, the new
pod may have a different IP address than before. That's why Kubernetes
introduced the concept of Services to address this issue.

When accessing a Service, whether it's a pod through Cluster IP +
TargetPort or using Node node IP + NodePort, the traffic is all
redirected by the Node node's iptables rules to the kube-proxy, which
listens on the Service's proxy port.

Once kube-proxy receives the Service's access request, it forwards it to
the backend pods based on the load balancing strategy. Kube-proxy not
only maintains iptables but also monitors the service's ports and
performs load balancing.

### Pod to Service

This is how communication from pod1 to service2 running on pod2 in
different nodes works:

{{< figure src="image23.gif" title="p2s" >}}

Pod to service

### Service to Pod

This is how communication from service2 to pod1 works:

{{< figure src="image24.gif" title="s2p" >}}

Service to pod

In summary, service discovery is achieved through Service resources that
provide stable IP addresses and DNS names for accessing pods. These
services manage load balancing and traffic routing to destination pods.
Kube-proxy generates and maintains the mapping table between services
and pod:port pairs, with options for iptables or IPVS load balancing
modes to suit performance and efficiency requirements.

## Between Internet and Service

In Kubernetes, communication between the internet and a Service is
typically achieved through the use of an Ingress resource. An Ingress is
an API object that manages external access to the services in a cluster,
typically for HTTP traffic. It acts as a layer of abstraction for
handling external traffic and provides features like load balancing, SSL
termination, and name-based virtual hosting.

Here's how Kubernetes achieves communication between the internet and a
Service using an Ingress:

- Ingress Resource: First, you create an Ingress resource that defines
  the rules for routing external traffic to your services. This includes
  specifying the hostnames, paths, and backend services to route the
  traffic to.

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  rules:
    - host: example.com        # Hostname for which traffic is routed
      http:
        paths:
          - path: /app1         # Path under the hostname
            pathType: Prefix    # Type of path matching
            backend:
              service:
                name: service1   # Backend service for this path
                port:
                  number: 80     # Port number for the backend service
          - path: /app2
            pathType: Prefix
            backend:
              service:
                name: service2
                port:
                  number: 8080
```

- Ingress Controller: To fulfill the Ingress resource, you need an
  Ingress controller. The Ingress controller is a component responsible
  for implementing the rules defined in the Ingress resource. There are
  various Ingress controllers available, such as Nginx Ingress
  Controller, Traefik, and others. You deploy the Ingress controller in
  your cluster.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx  # You can choose the namespace
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-ingress-controller
  template:
    metadata:
      labels:
        app: nginx-ingress-controller
    spec:
      containers:
        - name: nginx-ingress-controller
          image: k8s.gcr.io/ingress-nginx/nginx-ingress-controller:<controller-version>
          args:
            - /nginx-ingress-controller
            - --configmap=$(POD_NAMESPACE)/nginx-configuration
            - --tcp-services-configmap=$(POD_NAMESPACE)/tcp-services
            - --udp-services-configmap=$(POD_NAMESPACE)/udp-services
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-ingress-controller
  namespace: ingress-nginx  # Use the same namespace as the Deployment
spec:
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
  selector:
    app: nginx-ingress-controller
```

- External Access: Once the Ingress controller is running, it watches
  for changes in the Ingress resources and configures itself
  accordingly. When external traffic arrives at the cluster, it is first
  intercepted by the Ingress controller, which then uses the rules from
  the Ingress resource to route the traffic to the appropriate backend
  service.

- Load Balancing: If load balancing is required, the Ingress controller
  can distribute incoming traffic to multiple pods of the backend
  service, ensuring scalability and high availability.

- SSL Termination: If SSL termination is specified in the Ingress
  resource, the Ingress controller can handle SSL decryption and
  encryption, allowing secure communication with the backend service.

- Name-Based Virtual Hosting: Ingress can also support name-based
  virtual hosting, allowing you to host multiple websites or services on
  the same IP address and port, differentiating them based on the
  hostname specified in the Ingress rules.

# Conclusion

Kubernetes infrastructure is designed to be highly compartmentalized. A
highly structured plan for communication is important, as namespaces,
containers, and pods are meant to keep components separate from each
other.

There are three parallel network layers in Kubernetes:

- Node-to-node communication: Physical or virtual host routing (underlay
  routing).

- Pod-to-pod communication: overlay network or other network plugins to
  achieve it.

- Service-to-service communication: use kube-proxy and iptables rules to
  perform load balancing and route traffic to the correct endpoints
  (pods) based on labels and selectors.

Service Loadbalancer over Pod Network over Node Network = Kubernetes
Network

In more detail, it can be described as follows: 'Service Loadbalancer
over (Container Shared Namespace Network(Localhost) on Pod Network) over
Node Network = Kubernetes Network.'

Whether it's Docker or Kubernetes, they both run on Linux. Linux can be
considered the fundamental foundation. Before Kubernetes existed,
developers used Linux networking architecture to fulfill advanced
requirements. Kubernetes, when introduced, was built upon Linux
networking principles.

Hence, gaining an overview of the counterparts of Linux networking
components in the Kubernetes networking architecture is valuable. Many
engineers with years of Linux experience will find this format
convenient for understanding how Kubernetes implements complex
networking. Please refer to the table below for details.

<table>
<colgroup>
<col style="width: 49%" />
<col style="width: 50%" />
</colgroup>
<tbody>
<tr class="odd">
<td><strong>Linux Networking</strong></td>
<td><strong>Kubernetes Networking<br />
</strong></td>
</tr>
<tr class="even">
<td>Network Namespace</td>
<td><p>Node in Root Network Namespace</p>
<p>Pod in Pod Network Namespace</p></td>
</tr>
<tr class="odd">
<td>Network Interface (Physical+Virtual)</td>
<td>CNI(Container Network Interface)</td>
</tr>
<tr class="even">
<td>Veth Pair</td>
<td>Veth Pair on pods</td>
</tr>
<tr class="odd">
<td>Bridge Interface</td>
<td>cni0 (on Flannel plugin)</td>
</tr>
<tr class="even">
<td>NULL</td>
<td><p>CRI(Container Runtime Interface)<br />
(Docker/containerd/CRI-O</p>
<p>Mirantis Container Runtime)</p></td>
</tr>
<tr class="odd">
<td>NULL</td>
<td>kube-apiserver</td>
</tr>
<tr class="even">
<td>NULL</td>
<td>kube-controller-manager</td>
</tr>
<tr class="odd">
<td>NULL</td>
<td>etcd</td>
</tr>
<tr class="even">
<td>systemd/Container Runtime/Process Supervisor/cgroup/namespace</td>
<td>kubelet</td>
</tr>
<tr class="odd">
<td>iptables mode/IPVS mode/routing and network policy /proxy
server</td>
<td>kube-proxy</td>
</tr>
<tr class="even">
<td>IPv4 and IPv6 Protocol Stack</td>
<td>SingleStack/PreferDualStack/RequireDualStack/ipFamilies</td>
</tr>
<tr class="odd">
<td>DNS</td>
<td>KubeDNS(old), CoreDNS (k8s version&gt;1.13)</td>
</tr>
<tr class="even">
<td>Load Balancer(nginx,haproxy)</td>
<td>Service LoadBalancer/Service ClusterIP(L4)/Ingress(L7)</td>
</tr>
<tr class="odd">
<td>Security (Netfilter/iptables)</td>
<td>NetworkPolicy</td>
</tr>
<tr class="even">
<td>Logs</td>
<td>Fluentd/Logtail</td>
</tr>
</tbody>
</table>

As we conclude our comprehensive exploration of Kubernetes' networking
architecture. I hope you've found the information valuable and
insightful. If you have any questions or come across any inaccuracies in
the content, please don't hesitate to provide feedback on my personal
[<u>website</u>](https://xpetersue.com).
