# Overall architecture of the system

Kubernetes is a very complex system, and the setup reflects that.

## Hosts
The cluster described in this document will consist of ten individual servers,
or hosts in Kubernetes parlance:

- One certificate authority
- An etcd cluster consisting of three servers. For production, especially in
larger systems, five servers are recommended.
- A front-end server/load balancer
- Two redundant control plane systems. You should be able to add as many as you
want.
- Two worker nodes. You should be able to add as many as you want.
- An admin node. This will only have kubectl set up and configured. In a
real-life scenario, this would be your desktop workstation.

The control plane and worker node systems need to 2GB of memory, all other
host need 1 GB. Of course, in production you would probably want to give them
more memory.

Each host will need one NIC with an IP address that can reach all the other
hosts without NAT in between.

Each host must also have a DNS entry for this IP address. Alternatively, you
can maintain a /etc/hosts file on each host.

## Networks

Kubernetes networking is a beast. This cluster requires four networks:

### The vagrant/VirtualBox network

This is only an artifact of using Vagrant together with VirtualBox and has
nothing to do with Kubernetes. If you are using Vagrant with a different
hypervisor, or if you are using VirtualBox without Vagrant, you will not have
this network at all.

When it exists, this will be the default adapter in your virtual machine, and
have the IP address 10.0.2.15. As you will see, this extra NIC will give us
some grief as Kubernetes and the other components assume that the default NIC
can be used as the infrastructure NIC. It can't because this NIC cannot talk to
any of the other virtual machines.

### The infrastructure network

This is the usual physical network that every system has. If your system has a
single NIC, that is the infrastructure network. In the case of Vagrant/VirtualBox,
it will generally be the second NIC in the system, not the default one. On my
system, it works out to always be enp0s8, but this may not be the case for you.

To configure this NIC in a Vagrantfile, you must add the following line to each
instance (only for VirtualBox - you do not do this for libvirt):

    instance.vm.network "private_network", type: "dhcp"

### The pod network

This is a virtual network that the PODs use to talk to each
other. It is initially not tied to any NIC. As you build the cluster, the CNI
plugin you choose (we will be using flannel) will create a virtual NIC on each
host.

The IP address assignment in this virtual NIC originates in the API server. We
will get to the details of the API server later; it is part of the Kubernetes
Control Plane. You specify a large subnet (such as a /16 or even a /12 if your
cluster is going to be large) that the API server will then subdivide into /24s.
Each of these /24s will be assigned to one host, and one of the IP addresses,
usually the first one, will be the IP address for the host's virtual NIC. Each
pod will receive another IP address from this /24.

Pitfalls
---

Some network plugins, such as flannel, include hardcoded references to the
pod network IP address range. In that case, always use 10.244.0.0/16, or you can
edit the flannel.yaml file (we'll get to that later) to change the hard-coded
reference.

The IP address range must not exist anywhere on your internal network.

### The service network

This is a second virtual network. It is used to access Kubernetes services.
You will specify the IP address range, such as 10.32.0.0/24. Each service will
get an IP address in this range. The Kubernetes controller manager assigns these
IP addresses. Other than specifying the range, you don't really have to configure
much.

Note that some services will insist on specific hard-coded IP addresses. For
instance, CoreDNS will always be at 10.32.0.10.

One thing that makes the service network interesting, and sometimes hard to
troubleshoot, is that these IP addresses are never assigned to any NIC. Instead,
they will be mapped to actual IP addresses.

How this happens is fiendishly complicated to explain, because it can happen in
many different ways. The simplest scenario is that the kube-proxy daemon on each
worker node creates iptable/netfilter entries that translate these virtual
service IP addresses to actual physical IP addresses. But there are many other
mechanisms that can do the same thing.

