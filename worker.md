# Kubernetes the hard way - Worker Nodes

# Overview

The worker nodes are where the actual work is done, where the containers are
running. There are many different options for setting up worker nodes.
Generally, there will be one kubelet service running, as well as a networking
component and a container runtime.

The networking component we use is kube-proxy. This is the "standard" way to
use Kubernetes, but there are many efforts to use different approaches.

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

