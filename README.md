# Kubernetes the Hard Way

Another version of the learning site Kubernetes The Hard Way

Thanks goes to Kelsey Hightower for the [original Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) site,
and to the many others who have since stepped in his footsteps and created more
similar sites for various situations.

So why create yet another one?
---

This version will focus on some of the aspects Kelsey Hightower simplified for
the sake of simplicity. In particular, the goal is to create a documentation
that will:

- Use bare metal, similar to
[Praqma's](https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md)
 version of Kubernetes the Hard Way. I found some of Kelsey Hightower's
documentation hard to follow because he relied heavily on Google's environment.
- create a full PKI. Thanks goes to [Mike Newswanger](https://www.mikenewswanger.com/posts/2018/kubernetes-pki/)
- support certificate rollover.
- Use RBAC and avoid ABAC. Kelsey Hightower did this as well, but some other
versions don't.
- Focus more heavily on explaining the networking part, as this what I have been
struggling with the most.
- I will also try to explain each step.

My environment
---
The environment I am using to develop this is based on VirtualBox, Vagrant and
Ansible, but should hopefully work on any environment. I am not sharing the
complete Ansible role as this is proprietary and may not always make sense in
other environments, but I may include snippets, such as tasks and templates.

The cluster will run on CentOS 7. I do not use RHEL 7 here even though I have
access to it, because RedHat does not support any form of Kubernetes other than
their proprietary version OpenShift.

Prerequisites
---

To do this with virtual machines, you will need a very beefy host. I am using a
system with 32 GB of memory and a 4 core CPU. This appears to be the minimum
that will work.

If you have a smaller system, you may eliminate the separate systems for the
certificate authority, etcd, and the front end server, and use a single control
plane server. You will lose some of the benefits such as high availability, but
it should work.

Request for comments
---------

My goal is to learn to set up a production-quality cluster, but this document
will not be perfect, and I do not recommend actually putting the resulting
system into production. Please feel free to submit suggestions for improvement
as an issue, or as a pull request.

The parts of this document
---

- [Architecture](./architecture.md)
- [Public Key Infrastructure](./pki.md)
- [The etcd cluster](./etcd.md)
- [The front end server/load balancer](./front.md)
- [Control Plane](./cp.md)
- [The worker nodes](./worker.md)
- [Cluster-wide configuration](./cluster.md)

Resources
---------

### Kubernetes Basics
- https://github.com/kelseyhightower/kubernetes-the-hard-way
- https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md
- https://github.com/kubernetes/kops/blob/master/docs/install.md
- https://nixaid.com/deploying-kubernetes-cluster-from-scratch/

### PKI and Security
- https://www.mikenewswanger.com/posts/2018/kubernetes-pki/
- https://github.com/cloudflare/cfssl
- https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/
- https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster
- https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7

### RBAC
- https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/
- https://github.com/uruddarraju/kubernetes-rbac-policies

### Helm and other basic applications
- https://helm.sh/docs/using_helm/#quickstart-guide
- https://github.com/kubernetes/dashboard

