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

### Setting up the CA server.

The plan is to run the certificate authority configured for remote access,
listening on port 8888. Clients have to authenticate with a preshared token.
The token is generated as a random 16-digit hexadecimal number. I use the
following code in Ansible to generate it early in my playbook; it will generate
one token for client, peer and server certificates, and a different token for
the intermediate-CA certificates that would be even more dangerous if it falls
in unauthorized hands. The token must of course be known to both the CA server,
and to any host that has a need to obtain a certificate.

```
- name: Generate CA default auth key
  shell: "head -c 16 /dev/urandom | od -An -t x | tr -d ' '"
  register: r_defaultkey
  run_once: True

- name: Generate CA intermediate CA auth key
  shell: "head -c 16 /dev/urandom | od -An -t x | tr -d ' '"
  register: r_intermediatekey
  run_once: True
```

The actual key will be in r_defaultkey.stdout and r_intermediatekey.stdout,
respectively. I then use set_fact to extract the key for later:

```
- name: Set convenience variables
  set_fact:
    signing_auth_key:         "{{ r_intermediatekey.stdout }}"
    intermediateca_auth_key:  "{{ r_defaultkey.stdout }}"
```

Note the run_once: True attribute. This is critical; without it, each host would
see a different token, and that of course would not work. run_once causes the
key to be generated on only one system, rather than all hosts (it doesn't
matter which one). The variable will still be set on all hosts, but to the same
value that was generated on the single host that the task was actually run on.

To configure the CA server:

All configuration will be in a directory /etc/cfssl. This directory must be
accessible only to the cfssl user (and root, obviously).

Create a user to run the certificate authority. The user will be a system user
(-r), have /etc/cfssl as home directory and does not allow interactive logins.
Adding this user will also create the /etc/cfssl directory, and set the
permissions on it correctly.

    groupadd cfssl
    useradd -d /etc/cfssl -s /bin/nologin -g cfssl -r cfssl

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

Put the following into /etc/cfssl/ca-config.json

```
{
    "auth_keys": {
        "default": {
            "key": "f9e288c83734f9b9364729ef6f2a5dbf",
            "type": "standard"
        },
        "intermediateca": {
            "key": "50b81e58c40e1098dcb9947db260dc52",
            "type": "standard"
        }
    },
    "signing": {
        "default": {
            "expiry": "72h"
        },
        "profiles": {
            "client": {
                "auth_key": "default",
                "expiry": "72h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "client auth"
                ]
            },
            "intermediateca": {
                "auth_key": "intermediateca",
                "ca_constraint": {
                    "is_ca": true,
                    "max_path_len": 0,
                    "max_path_len_zero": true
                },
                "expiry": "87600h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "cert sign",
                    "crl sign"
                ]
            },
            "peer": {
                "auth_key": "default",
                "expiry": "72h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            },
            "server": {
                "auth_key": "default",
                "expiry": "72h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth"
                ]
            }
        }
    }
}
```

The auth_keys section includes both of our authentication keys.

The signing section lists four profiles, along with the certificate attributes
that each one should have. For each profile, it also lists which auth_key can
be used. If you use the wrong auth key later, you will get an authorization
denied error.

Create the CSR for the certificate authority certificate. Put the following into
/tmp/$$.csr. Note that this CSR does not include any key information, so it is
not particularly sensitive. The contents of the DN attributes are not relevant,
I would set them to company or department names.

```
{
  "CN": "kubernetes-the-hard-way",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Diego",
      "O": "My Company",
      "OU": "IT Department",
      "ST": "California"
    }
  ]
}
```

Create the certificate authority certificate:

```
cd /etc/cfssl
/usr/local/sbin/cfssl gencert -initca /tmp/$$.csr | \
        /usr/local/sbin/cfssljson -bare ca
chown cfssl ca-key.pem
```

TODO: explain certificate validity.

It is now a good idea to delete the /tmp/$$.csr file.

Set up remote access using the multirootca utility. This utility needs its own
server certificate, since it uses TLS to allow clients to request certificates.

So we need to set up another CSR. Since this is a standard server CA, it must
contain the FQDN under which the server will be reached in the CN.

```
{
  "CN": "myca.example.com",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "US",
      "L": "San Diego",
      "O": "My Company",
      "OU": "IT Department",
      "ST": "California"
    }
  ]
}
```

This, and the original CA certificate, is the only one issued directly by cfssl.
All the future certificates will be issued remotely. To issue this certificate,
run the following:

```
cd /etc/cfssl
/usr/local/sbin/cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=server /tmp/$$.csr | \
/usr/local/sbin/cfssljson -bare server
```

You have to re-run this every time the certificate expires, and then restart
the cfssl service that we will be setting up next. Since the certificate
will, by design, expire every few days, you may want to use a cron job to do
this.

The multirootca utility needs a configuration file. Place the following in
/etc/cfssl/multiroot-profile.ini. The ip address list is a comma-separated
whitelist that decides which IP is allowed to request certificates. It should
contain either the subnet where all your hosts are located, or ideally a list
of all IP addresses the etcd, cp, worker and admin hosts, each with the /32
suffix. Also include 127.0.0.1/32.

```
[default]
private = file://ca-key.pem
certificate = ca.pem
config = ca-config.json
nets = <ip address list>
```

Set up cfssl as a service. Create the following systemd unit file in
/etc/systemd/system/cfssl.service:

```
[Unit]
Description=CFSSL PKI Certificate Authority
After=network.target
Documentation=https://blog.cloudflare.com/introducing-cfssl/

[Service]
User=cfssl
Type=simple
ExecStart=/usr/local/sbin/multirootca \
    -a 0.0.0.0:8888 \
    -l default \
    -roots /etc/cfssl/multiroot-profile.ini \
    -tls-cert /etc/cfssl/server.pem \
    -tls-key /etc/cfssl/server-key.pem
Restart=on-failure
WorkingDirectory=/etc/cfssl

[Install]
WantedBy=multi-user.target
```

Don't forget to use daemon-reload before trying to enable and start the service:

```
    systemctl daemon-reload
    systemctl cfssl enable
    systemctl cfssl start
```

You should at this point be able to establish an HTTPS connection on port 8888.

Finally, retrieve a copy of the /etc/cfssl/ca.pem file - this is the CA's
certificate. You need to distribute this file throughout your infrastructure
so the various hosts can trust this CA.

### Creating and using the certificates

Since there are different uses for the certificates, they need to have different
attributes.

- Server certificates must include all possible host names and IP addresses
under which a server can be reached.
- client certificates for anything Kubernetes-related will be used with user
names and group names. The user names go in the CN attribute, the group name
goes in the O attribute.

Copy the cfssl and cfssljson utilities to /usr/local/bin on each of the clients.
The clients do not need the multirootca utility.

On each client, put the certificate authority's certificate into
/etc/pki/tls/certs/ca.pem.

Next, create a PKI directory for our system. On most of our servers, this will
be /etc/kubernetes/pki . On the etcd cluster, it will be /etc/etcd/pki

TODO: finish this section.

### Secure the server:

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

