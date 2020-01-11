# Kubernetes the hard way - Worker Nodes

## Overview

The worker nodes are where the actual work is done, where the containers are
running. There are many different options for setting up worker nodes.
Generally, there will be one kubelet service running, as well as a networking
component and a container runtime.

The networking component we use is kube-proxy. This is the "standard" way to
use Kubernetes, but there are many efforts to use different approaches.

All control plane nodes are also worker nodes, although they generally do not
run any workloads. This is handled through the Kubernetes feature of "tainting".
Therefore, perform the following tasks on cp1, cp2, worker1
and worker2.

## Components

### The kubelet service

The kubelet service communicates with the API server. It also instructs the
container runtime which images to load and which containers to create and run.
The kubelet service is part of Kubernetes.

### The container runtime

The container runtime is a separate project. There are many different options.
Among the options are Docker, containerd and cri-o.

While Docker is the oldest and best-known container runtime, it is rarely used
with Kubernetes today, because the docker daemon must run as root, and has far
more functionality than Kubernetes requires.

We are using containerd in this example.

cri-o is getting very popular with Kubernetes, but requires compilation and is
harder to set up.

### Networking

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

    mkdir -p /etc/kubernetes/pki /var/run/kubernetes /etc/cni/net.d /opt/cni/bin
    groupadd kubernetes
    useradd -d /etc/kubernetes -s /bin/nologin -g kubernetes -r kubernetes
    chown -R kubernetes:kubernetes /etc/kubernetes /var/run/kubernetes
    chmod 0755 /etc/kubernetes /var/run/kubernetes
    chmod 0700 /etc/kubernetes/pki

Remove firewalld. The kube-proxy daemon will set up various iptables rules. It
is possible, but not trivial, for those iptable rules to coexist with any rules
firewalld may have added to the node.

Download the binaries you need and place them in /usr/local/sbin

    curl https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kubelet -o /usr/local/sbin/kubelet
    curl https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/kube-proxy -o /usr/local/sbin/kube-proxy
    curl https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.15.0/crictl-v1.15.0-linux-amd64.tar.gz -o /tmp/crictl.tar.gz
    tar xfz /tmp/crictl.tar.gz -C /usr/local/sbin --strip-components=1
    curl https://github.com/containernetworking/plugins/releases/download/v0.8.2/cni-plugins-linux-amd64-v0.8.2.tgz -o /tmp/cni.tgz
    tar xfz /tmp/cni.tgz -C /opt/cni/bin --strip-components=1
    rm /tmp/crictl.tar.gz /tmp/cni.tgz
    chmod +x /usr/local/sbin/{kubelet,kube-proxy,crictl}

For containerd, we install an RPM. It is located in the official Docker repository.

    yum install containerd.io

## Set up the Container runtime

The container runtime is responsible for actually executing the containers. We are
using containerd, which is part of Docker. Another option is cri-o. cri-o is preferred
because it is more light-weight, but is currently a bit harder to install.

Obtain the gid of the kubernetes group:

    getent group kubernetes

Edit the containerd configuration file in /etc/containerd/config.toml

    [grpc]
      address = "/var/run/containerd/containerd.sock"
      uid=0
      gid=<kubernetes user group ID>
    [plugins]
      [plugins.cri.containerd]
        snapshotter = "overlayfs"
        [plugins.cri.containerd.default_runtime]
          runtime_type = "io.containerd.runtime.v1.linux"
          runtime_engine = "/bin/runc"
          runtime_root = ""

Now enable and start (or restart) the containerd service:

    systemctl enable containerd
    systemctl restart containerd

## Set up the kubelet server

The kubelet server receives instructions from the API server, and executes them
by using the container runtime to start or stop containers, retrieve log files,
and whatever other tasks may come up.

It requires two certificates:

- A client certificate to connect to the API server.
- A server certificate to allow the API server to retrieve information (log
  files etc.)

The client certificate can in some cases be automatically created and renewed.
We will be using this bootstrap mechanisms. Note: this requires Kubernetes 1.16
or later.

The server certificate has to be explicitly created with the following parameters:
CN: system:node:<fqdn>
O: system:nodes
Hosts:
  <node IP address>
  <node FQDN>
.

To set up the kubelet server:

- Create a CNI plugin configuration file for the loopback interface. Edit the
  file /etc/cni/net.d/99-loopback.conf with the following content:

```
{
    "cniVersion": "0.3.1",
    "name": "lo",
    "type": "loopback"
}
```

- Create the kubelet bootstrap kubeconfig file. The certificate-authority-data
  is a concatenation of all the certificate authorities - the main CA, as well
  as the intermediate CAs on the control plane nodes.
  The token must be added to the API server as described in the section on the
  control plane. The token can be any pair of random hexadecimal values, where
  the first one is six digits, and the last one is 16 digits. The first part is
  considered public and is used to identify the token; the second part should
  be considered private and confidential.

  The bootstrap config file should have the following content:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUR5RENDQXJDZ0F3SUJBZ0lVYUlSa2ZVNllaRzM3ZDJTWHAvanlseGZCd2JFd0RRWUpLb1pJaHZjTkFRRUwKQlFBd2ZERUxNQWtHQTFVRUJoTUNWVk14RXpBUkJnTlZCQWdUQ2tOaGJHbG1iM0p1YVdFeEVqQVFCZ05WQkFjVApDVk5oYmlCRWFXVm5iekVnTUI0R0ExVUVDaE1YVlc1cGRtVnljMmwwZVNCdlppQlRZVzRnUkdsbFoyOHhFVEFQCkJnTlZCQXNUQ0VsVVV5Qk9TVk5UTVE4d0RRWURWUVFEREFaVlUwUmZRMEV3SGhjTk1Ua3hNakU0TVRneU1UQXcKV2hjTk1qUXhNakUyTVRneU1UQXdXakI4TVFzd0NRWURWUVFHRXdKVlV6RVRNQkVHQTFVRUNCTUtRMkZzYVdadgpjbTVwWVRFU01CQUdBMVVFQnhNSlUyRnVJRVJwWldkdk1TQXdIZ1lEVlFRS0V4ZFZibWwyWlhKemFYUjVJRzltCklGTmhiaUJFYVdWbmJ6RVJNQThHQTFVRUN4TUlTVlJUSUU1SlUxTXhEekFOQmdOVkJBTU1CbFZUUkY5RFFUQ0MKQVNJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dFUEFEQ0NBUW9DZ2dFQkFMdWJJc09ReGdsV2wrUlhiRFcxeFZiVAp6ckUzNnRwSWNHODNuYUV5NUFjN1hnOTlKMENDMDQ3eGh2Y1V5OXA4cmV5TTJwR0VZOVg3Q3l1ZVBONTRMTzRjCjFmTUREajNkaDlWSmFHTEZhU1JPcm13N3J4VVlXTnU2djVzTGd0elR2SWFGMTJaTFZrRUdNMFFob2U5ME50RzYKYjQvT1VRYlBqMzdEeEJuM2tES2dibGpIS1dNRXdwZDB6Q2M5SWRNWFJYVzV1VEI3WEIyb2FIK0pLQlptV0o0NApGOEZQZ0s0MmlMQUcraEREWDN4UDE4eTVHZmtPZHJqRjBXRmZZOXliNUV6aXZEdGk1b3E4amovRFdUUnZhaSs5ClNOUjVoUlp4TmR6elM1Mm1WeGExeEdOMFhwUEVIK0dTaWpqWTJZRjRKekE2eEhudUY5aDRoRWdUekdJNHhXY0MKQXdFQUFhTkNNRUF3RGdZRFZSMFBBUUgvQkFRREFnRUdNQThHQTFVZEV3RUIvd1FGTUFNQkFmOHdIUVlEVlIwTwpCQllFRk0yRFpUcWJuR1RuRllmOExTeXdQSkRiSzlXZE1BMEdDU3FHU0liM0RRRUJDd1VBQTRJQkFRQ0NxdTM1CndqdmtkRFZYRVFTUFRydnJEeWc2dSt5VGZONFRIQmkxOFBsRy9adnVMMVd4NVRNVlA2WHdjY1BCU1dreGdZOG4KTmFTSDF6bnNRQlp2c0svSm9JOWRqSkJRU2twTHcyNWw0cGoyMzFPVklMdmVNSjZYajFqdUZ3R3dNMU9ad0IyRQppRUpLRGV2am5wcDZFVzdGcFdjYzN1K2ZsZHpLOFRuWkU5MzBSMVVRTmpEcFZROTZoR21MQ3JxeHk2a1lCNkxLCk5YTFZzV2Z6VTE5cXR1UTAyanA3Zmh5cGdXVVJ0SENJbHVoaUlSL2VmTlFuQzlaUVJoaDBhcGE4OWxaNk4xWTEKMm9KZnJhUlJ1MldLckN0NFRSUndxZStqeFF6ZHlqeEpKL2t1cGpac2JYaEJMQXJ3bzJTcFpqb3NqUjVRMjhKNgo4SUFWOGttRzRPQmJXRVduCi0tLS0tRU5EIENFUlRJRklDQVRFLS0tLS0K
    server: https://front.vagrant:6443
  name: vagtestcluster
contexts:
- context:
    cluster: vagtestcluster
    user: kubelet-bootstrap
  name: default
current-context: default
kind: Config
preferences: {}
users:
- name: kubelet-bootstrap
  user:
    token: 002439.25f718816c0c9343
```

Create the /etc/systemd/system/kubelet.service unit file:

( You may need to add --node-ip=<ip address> if your node has multiple NICs)

```
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
# User=kubernetes
ExecStart=/usr/local/sbin/kubelet \
    --config=/etc/kubernetes/kubelet-config.yaml \
    --cert-dir=/etc/kubernetes/pki \
    --container-runtime=remote \
    --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \
    --image-pull-progress-deadline=2m \
    --kubeconfig=/etc/kubernetes/kubelet.kubeconfig \
    --bootstrap-kubeconfig=/etc/kubernetes/bootstrap.kubeconfig \
    --network-plugin=cni \
    --cni-conf-dir=/etc/cni/net.d \
    --register-node=true \
    --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

```

## Set up the kube-proxy service

The kube-proxy service is responsible for setting up the pod and service
networks.i It needs a client certificate to contact the API server

The client certificate has to be explicitly created with the following parameters
CN: system:kube-proxy
O: system:node-proxier
Hosts:
  <node IP address>
  <node FQDN>

Create a kubeconfig file with this certificate as /etc/kubernetes/kube-proxy.kubeconfig

Install the tools for ipvs

    yum install ipvsadm

Create the kube-proxy configuration file as /etc/kubernetes/kubeproxy-config.yaml :

```
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/etc/kubernetes/kube-proxy.kubeconfig"
mode: "ipvs"
clusterCIDR: "10.244.0.0/16"
```

Create the service unit file as /etc/systemd/system/kube-proxy.service:

```
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/sbin/kube-proxy \
--config=/etc/kubernetes/kubeproxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Now enable and start (or restart) the kubelet and kube-proxy services:

    systemctl daemon-reload
    systemctl enable kubelet kube-proxy
    systemctl restart kubelet kube-proxy

Next: [Control Plane](./cp.md)

