# Setting up the Public Key Infrastructure

The PKI is based on Cloudflare's excellent cfssl tool.

Older versions of this tool are available for download from cloudflare's
Web site, but to get the latest version, you have to compile from scratch.

Fortunately, this is very easy to do. You will need a working Go language
environment on your workstation (not on one of the cluster systems). On
RedHat-like systems, simply install the following RPMs: golang golang-bin.

Run the following script in an empty directory. It will download the cfssl
source code, compile it for you and copy the three files you need into the root
of the same empty directory.

```
export GOPATH=${PWD}
git clone https://github.com/cloudflare/cfssl.git $GOPATH/src/github.com/cloudflare/cfssl
pushd $GOPATH/src/github.com/cloudflare/cfssl
make
popd

cp $GOPATH/src/github.com/cloudflare/cfssl/bin/{cfssl,cfssljson,multirootca} ${PWD}
```

Copy all three of these files to /usr/local/sbin on the following hosts:

- ca
- etcd
- cp
- worker
- admin

The front-end server does not need any certificates.

On the CA server:

Create a user to run the certificate authority. The user will be a system user
(-r), have /etc/cfssl as home directory and does not allow interactive logins.

    useradd -d /etc/cfssl -s /bin/nologin -r cfssl

Create a configuration file for the certificate authority. Throughout the
cluster, we will need five different types of certificate:

- The certificate for the CA itself.
- Server certificates. This is essentially the familiar type of SSL certificate
that HTTP servers have. Most of the Kubernetes components will have one of these.
- Client certificates. This takes the place of user name and password, both when
Kubernetes components need to talk to each other, and also when a human needs to
administer the client certificates. Most of the client certificates will later be
wrapped into kubeconfig files.
- Peer certificates. In a few cases, Kubernetes components need to communicate
bidirectionally, where either end can act as a server or a client.
- Intermediate CA certificates. Because we want to allow the worker nodes to
automatically obtain certificates, the API server/Cluster Manager combination
must be configured with a CA certificate. We set them up as intermediate CAs.

```
TODO
```

Create the certificate authority certificate:

```
TODO
```

Set up remote access using the multirootca utility:

```
TODO
```

Set up multirootca as a service. Create the following systemd unit file in
/etc/systemd/system/multirootca.service:

```
TODO
```

Secure the server:

As with any CA, you want to secure the server to the best of your abilities.
Generally, hardening a server is beyond the scope of this document, but here are
a few key points:

- Use the server for no other purpose than as a CA server.
- Block all ports except those you absolutely need. In our case, this is SSH and
port 8888.
- Block access from all IP addresses except those that need access. SSH should
only be allowed from those systems that need to administer the CA. Port 8888
should only be allowed from etcd servers, worker nodes, control plane nodes, and
from administrator workstations.
- Be careful how you distribute the CA authentication tokens. In particular, the
token for intermediate CA certificates should only be available to the control
plane servers.
- If you are using a configuration management tool such as Ansible:
  - protect the authentication tokens through something like a vault.
  - turn off logging for any tasks that may expose the authentication tokens.

Personally, I create new authentication tokens on each Ansible run. The same
run will then also distribute the new tokens to all the hosts that need them.
This is a simple mechanism for frequently changing tokens, although it could
conceivably lead to a race condition in rare situations.

