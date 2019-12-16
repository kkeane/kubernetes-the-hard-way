# The Control Plane

The control plane includes everything that manages the cluster. There are many
different ways to set up a control plane. We are setting up a mostly bare-bones
version here.

## Components

The control plane fundamentally has three components: the API server, the
controller manager, and the scheduler. These three servers communicate with
each other extensively. The outside world will only communicate with the API
server.

It is easily possible, and highly recommended, to set up multiple control plane
servers for high availability. In that case, treat the three services as a unit,
and set up several such units individually. Each such triad should only
communicate with its own peers, but not with another triad. Overall, they will
communicate only via shared access to the etcd cluster.

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

    for f in kube-apiserver kube-controller-manager kube-scheduler kubelet
    do
       curl https://storage.googleapis.com/kubernetes-release/release/v1.16.2/bin/linux/amd64/${file} -o /usr/local/sbin/${file}
       chmod +x /usr/local/sbin/${file}
    done

Optionally, install the bash completion for Kubernetes (this requires
installing kubectl in addition to the binaries listed above. Note that kubectl
should be in /usr/local/bin rather than /usr/local/sbin)

    /usr/local/bin/kubectl completion bash > /etc/bash_completion.d/kubectl
    chmod +x /etc/bash_completion.d/kubectl

## Set up the API server daemon

This daemon requires several certificates. All certificates should be owned
by user kubernetes.

- An etcd client certificate. This is used to connect to the etcd cluster.
  There are no special requirements for this certificate. The etcd daemon
  should accept any client certificate, as long as it was issued by our
  certificate authority.

- An API server server certificate. This is used to allow internal client
  software (kubectl, kube-controller-manager, kube-scheduler), as well as
  kubectl, to connect to the API server.
  This certificate must be named with the FQDN of the node you are setting up.
  This certificate must include the following Subject Alternate Names:
  - The external name of the cluster (i.e., the name of the front end load
    balancer)
  - The IP address of the control plane node.
  - The FQDN of the control plane node.
  - The IP address of the API servers on the service network (10.32.0.1, if
    you used the 10.32.0.0/24 service network).

- A client certificate to access kubelets. This is used, for instance, to
  retrieve logs from worker nodes.

- An intermediate certificate authority certificate. This will be used later
  by the kubelet service on the workers to bootstrap, and to renew some of
  its certificates.
  Once created, collect the intermediate CA certificates from all control
  plane nodes, and the CA certificate from the root, and concatenate all of
  them into a file /etc/pki/tls/certs/intermediate-cas.pem . Copy this file to
  all control plane and worker nodes.

Use the cluster's certificate authority, as described earlier, to obtain
these certificates.

You also need to create a service account key. This is a private key in
PEM format that is not associated with a certificate. Only create one
key, and copy it to all control plane nodes.

To generate such a key, run on the cp1 node:

    /bin/openssl genrsa >/etc/kubernetes/pki/cp1-sa-key.pem
    chown kubernetes:kubernetes /etc/kubernetes/pki/cp1-sa-key.pem
    chmod 0600 /etc/kubernetes/pki/cp1-sa-key.pem

Then **securely** copy the generated file to cp2 (and any other control plane
node, if you are setting up more), naming it cp2-sa-key.pem, and apply the same
permissions.

Next, set up the encryption configuration file. This file configures how data
is encrypted at rest, that is, within etcd.

**Important** Once created, do not lose this file. You will lose your cluster!
As long as your backups are securely stored, you should back up this file.

Create a file /etc/kubernetes/pki/encryption-config.yaml with ownership
kubernetes:kubernetes and permissions 0600, with the following content.

Change the secret. The secret must be exactly 32 bytes long, and
base64-encoded. You can generate such a key with the command:

    openssl rand 32 -base64

The resulting secret should have the same length as the sample shown
below.

```
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: '9uj3Y1FwHVQVRwrf5MrEOCojcZQ0/MygUV33mCr/pUg='
      - identity: {}
```

The secret must match on all of your control plane nodes.

Now you are ready to set up the API server systemd unit file. Create the
following file as /etc/systemd/system/kube-apiserver.service:

```
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
User=kubernetes
ExecStart=/usr/local/sbin/kube-apiserver \
  --advertise-address=172.28.128.231 \
  --allow-privileged=true \
  --apiserver-count=1 \
  --audit-log-maxage=30 \
  --audit-log-maxbackup=3 \
  --audit-log-maxsize=100 \
  --audit-log-path=/var/log/audit.log \
  --authorization-mode=Node,RBAC \
  --bind-address=0.0.0.0 \
  --client-ca-file=/etc/pki/tls/certs/intermediate-cas.pem \
  --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
  --enable-bootstrap-token-auth=true \
  --etcd-cafile=/etc/pki/tls/certs/ca.pem \
  --etcd-certfile=/etc/kubernetes/pki/cp1-etcdclient.pem \
  --etcd-keyfile=/etc/kubernetes/pki/cp1-etcdclient-key.pem \
  --etcd-servers=https://etcd1.vagrant:2379,https://etcd2.vagrant:2379,https://etcd3.vagrant:2379 \
  --event-ttl=1h \
  --external-hostname=front.vagrant \
  --encryption-provider-config=/etc/kubernetes/pki/encryption-config.yaml \
  --runtime-config=api/all \
  --service-account-key-file=/etc/kubernetes/pki/cp1-sa-key.pem \
  --service-cluster-ip-range=10.32.0.0/24 \
  --service-node-port-range=30000-32767 \
  --kubelet-certificate-authority=/etc/pki/tls/certs/intermediate-cas.pem \
  --kubelet-client-certificate=/etc/kubernetes/pki/cp1-apiserver-kubelet-client.pem \
  --kubelet-client-key=/etc/kubernetes/pki/cp1-apiserver-kubelet-client-key.pem \
  --kubelet-https=true \
  --tls-cert-file=/etc/kubernetes/pki/cp1-api.pem \
  --tls-private-key-file=/etc/kubernetes/pki/cp1-api-key.pem \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Here are the arguments in detail. The highlighted values probably need to be
changed for your situation.

- **--advertise-address: the infrastructure IP address of this node.**
- --allow-privileged: Allow privileged containers to be controlled from this
  API server. We need this for the flannel networking.
- **--apiserver-count: how many CP nodes do you plan on having?**
- --audit-log*: This configures where and how an audit log will be stored. An
  audit log lists details about when what type of event happened. These
  settings do not actually enable audit-logging; you would also have to
  provide an audit policy.
- --authorization-mode: Allows Node Authentication and RBAC authentication.
  Node authentication allows worker nodes to use a client certificate to
  connect to the API server and join the cluster. We are not actually using
  it, but it is a good idea to leave it enabled in case you ever run into a
  problem with bearer tokens. RBAC is the main authentication used within the
  cluster.
- --bind-address: which addresses should the API server listen on? 0.0.0.0
  means, everywhere
- --client-ca-file: certificates for all certificate authorities whose client
  certificates we will accept.
- --enable-admission-plugins: a list of plugins that modify how the API server
  handles authentication
- --enable-bootstrap-token-auth: this allows worker nodes to join using a token
  rather than a client certificate. It also enables automatic rollover of
  worker node certificates.
- --etcd-*: configures the connection to the etcd servers.
- **--etcd-servers: list all etcd servers here**
- --event-ttl: Keep events for 1 hour.
- --external-hostname: the host name of the load balancer to access the API
  server.
- --encryption-provider-config: configures the at-rest encryption of Kubernetes
  data in etcd.
- --runtime-config: describes which APIs should be available in this API
  server. api/all means, all apis.
- --service-account-key-file: the encryption key that will be used to encrypt
  and verify service tokens.
- --servuce-cluster-ip-range: the service network.
- --service-node-port-range: the port numbers that services can be visible as.
- --kubelet-certificate-authority: the CA (or CAs) that can issue the client
  certificates for the kubelet service on worker nodes.
- --kubelet-client-*: the client certificate the API server will use to
  authenticate when connecting to a kubelet.
- --kubelet-https: whether to use https for connections to the kubelets.
- --tls-*: the certificate and key the API server will present to anybody
  connecting.
- --v: the log level verbosity.

## The Controller Manager



Next: [The worker nodes](./worker.md)

