# Kubernetes the hard way - Worker Nodes

# Overview

The worker nodes are where the actual work is done, where the containers are
running. There are many different options for setting up worker nodes.
Generally, there will be one kubelet service running, as well as a networking
component and a container runtime.

The networking component we use is kube-proxy. This is the "standard" way to
use Kubernetes, but there are many efforts to use different approaches.

All control plane nodes are also worker nodes, although they generally do not
run any workloads. Therefore, perform the following tasks on cp1, cp2, worker1
and worker2.

## The kubelet service

The kubelet service communicates with the API server. It also instructs the
container runtime which images to load and which containers to create and run.
The kubelet service is part of Kubernetes.

## The container runtime

The container runtime is a separate project. There are many different options.
Among the options are Docker, containerd and cri-o.

While Docker is the oldest and best-known container runtime, it is rarely used
with Kubernetes today, because the docker daemon must run as root, and has far
more functionality than Kubernetes requires.

We are using containerd in this example.

cri-o is getting very popular with Kubernetes, but requires compilation and is
harder to set up.

## Networking

The purpose of the networking component is to create the service network, which
allows pods to talk to each other. It works closely together with the CNI
(Container Network Interface).

There are many different ways to accomplish this. A popular method is to use
the kube-proxy. Kube-proxy has several different modes of operation. It can
create iptables rules that masquerade the service IP addresses as Pod IP
addresses. It can also use ipvs.

## System setup

All steps need to be performed as user root.

Create a system user group and user kubernetes and create the required
directories:

    mkdir -p /etc/kubernetes/pki
    groupadd kubernetes
    useradd -d /etc/kubernetes -s /bin/nologin -g kubernetes -r kubernetes
    chown -R kubernetes:kubernetes /etc/kubernetes
    chmod 0755 /etc/kubernetes
    chmod 0700 /etc/kubernetes/pki

Download the binaries you need and place them in /usr/local/sbin

    curl https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubelet -o /usr/local/sbin/kubelet
    chmod +x /usr/local/sbin/kubelet
    


Next: [Control Plane](./cp.md)

