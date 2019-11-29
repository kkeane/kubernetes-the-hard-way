# Kubernetes the Hard Way

Another version of the learning site Kubernetes The Hard Way

Thanks goes to Kelsey Hightower for the (https://github.com/kelseyhightower/kubernetes-the-hard-way)[original Kubernetes The Hard Way] site,
and to the many others who have since stepped in his footsteps and created more
similar sites for various situations.

So why create yet another one?
---

This version will focus on some of the aspects Kelsey Hightower simplified for
the sake of simplicity. In particular, the goal is to create a documentation
that will:

- Use bare metal, similar to
(https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md)[Praqma's]
 version of Kubernetes the Hard Way. I found some of Kelsey Hightower's
documentation hard to follow because he relied heavily on Google's environment.
- create a full PKI. Thanks goes to (https://www.mikenewswanger.com/posts/2018/kubernetes-pki/)[Mike Newswanger]
- Use RBAC and avoid ABAC. Kelsey Hightower did this as well, but some other
versions don't.
- 

Resources
---------

## Kubernetes Basics
https://github.com/kelseyhightower/kubernetes-the-hard-way
https://github.com/Praqma/LearnKubernetes/blob/master/kamran/Kubernetes-The-Hard-Way-on-BareMetal.md
https://github.com/kubernetes/kops/blob/master/docs/install.md
https://nixaid.com/deploying-kubernetes-cluster-from-scratch/

# PKI and Security
https://www.mikenewswanger.com/posts/2018/kubernetes-pki/
https://github.com/cloudflare/cfssl
https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/
https://cloud.google.com/kubernetes-engine/docs/how-to/hardening-your-cluster
https://medium.com/@toddrosner/kubernetes-tls-bootstrapping-cf203776abc7

# RBAC
https://docs.bitnami.com/kubernetes/how-to/configure-rbac-in-your-kubernetes-cluster/
https://github.com/uruddarraju/kubernetes-rbac-policies

# Helm and other basic applications
https://helm.sh/docs/using_helm/#quickstart-guide
https://github.com/kubernetes/dashboard

