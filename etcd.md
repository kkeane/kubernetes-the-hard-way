# Setting up the etcd cluster

etcd is a fairly simple-to-set-up system even though it is very powerful.

On both RedHat 7 and CentOS 7, you can set it up by installing the etcd RPMs
from the Extras repository. In this document, we will instead set it up from
scratch.

## General information

etcd is a quorum-based distributed database. Ideally, all nodes should have all
of the data. Nodes can, and sometimes will, disagree about the actual data. In
that case, the data from the majority of nodes will be considered valid.

This means that an etcd cluster should always have an odd number of nodes. If
there are two nodes, and they disagree, neither can ever have a majority.
Similarly, if there are four nodes, and two sets of two agree on the data,
there will be a tie that cannot be resolved.

With an odd number of nodes in the cluster, this situation will not occur.

The database uses two ports. 2380 is used for communication between cluster
members, and 2379 is used by clients.

## Preparation steps

Get a list of host names and IP addresses for each member of the IP address
cluster. For instance, in our ase

- etcd1.vagrant 172.28.128.253
- etcd2.vagrant 172.28.128.252
- etcd3.vagrant 172.28.128.251

We will later use this list to build the cluster members list.

The remaining steps need to be completed on each node.

## Initial setup

- Open ports 2379 and 2380 in your firewall. 2380 only needs to be available
to the other etcd cluster nodes. 2379 needs to be available to the Kubernetes
management nodes as well.

- Create a data directory and a configuration directory, and a user and group

```
mkdir -p /var/lib/etcd /etc/etcd
groupadd etcd
useradd -d /etc/etcd -s /bin/nologin -g etcd -r etcd
chown etcd:etcd /var/lib/etcd /etc/etcd
chmod 0700 /var/lib/etcd
chmod 0755 etc/etcd
```

- Download the etcd binaries and extract etcd and etcdctl

The official URL to download etcd from is (adjust the version number as needed,
of course) https://github.com/etcd-io/etcd/releases/download/v3.4.3/etcd-v3.4.3-linux-amd64.tar.gz

Extract the tarball:

    tar xfz etcd-v3.4.3-linux-amd64.tar.gz

Copy the resulting files etcd and etcdctl to /usr/local/sbin

## Create certificates

Create a request config file as described in the PKI section, and place it in
/etc/pki/tls/private/request-config.json

Communication between the etcd servers, as well as communication between clients, needs to be
encrypted. Since communication between etcd nodes can go in either direction, we are using a
peer certificate that can act as both server and client.

On each etcd node, create the following CSR file (substituting the correct
host name in the CN) in /etc/etcd/pki/<nodename>-csr.json

```
{
    "CN": "etcd1",
    "hosts": [
        "etcd1.vagrant",
        "etcd2.vagrant",
        "etcd3.vagrant",
        "172.28.128.253",
        "172.28.128.252",
        "172.28.128.251"
    ],
    "key": {
        "algo": "rsa",
        "size": 2048
    },
    "names": [
        {
            "C": "US",
            "L": "San Diego",
            "O": "University of San Diego",
            "OU": "Kubernetes CA for vagtestcluster",
            "ST": "California"
        }
    ]
}
```

**Important**: Due to a bug in the etcd client library, the certificate must
contain the host names and IP addresses for all etcd nodes, not just the one
the certificate will be used on. The bug is that the client library will
always check the certificate against the name of the first etcd server, even
if it actually connects to one of the other ones.

Then use cfssl to generate the certificate:

```
cd /etc/etcd/pki
/usr/local/sbin/cfssl gencert \
  -config=/etc/pki/tls/private/request-config.json \
  -profile=peer etcd1-csr.json | \
/usr/local/sbin/cfssljson -bare etcd1
```

### Certificate rollover:

Re-run this command a few days before the certificate expires, and then
restart the etcd service.

Only restart one etcd node at a time to ensure high availability.

## Create the etcd configuration file

Create the following file in /etc/etcd/etcd.conf.yml. Update the
node name and IP addresses according to the node name and the
information you gathered earlier.

**Important**: if you are adding a node to an existing cluster,
you must change the entry "initial-cluster-state" to "existing".
You must also regenerate all SSH certificates on all nodes with
the complete list of host names and IP addresses.

```
name:     etcd1
data-dir: /var/lib/etcd
listen-peer-urls:   https://172.28.128.253:2380
listen-client-urls: https://0.0.0.0:2379
logger: zap
log-level: info

advertise-client-urls: https://etcd1.vagrant:2379
initial-advertise-peer-urls: https://etcd1.vagrant:2380
initial-cluster-token: etcd-cluster-0
initial-cluster: etcd1=https://etcd1.vagrant:2380,etcd2=https://etcd2.vagrant:2380,etcd3=https://etcd3.vagrant:2380
initial-cluster-state: new

client-transport-security:
  trusted-ca-file: /etc/pki/tls/certs/ca.pem
  cert-file: /etc/etcd/pki/etcd1.pem
  key-file: /etc/etcd/pki/etcd1-key.pem
  client-cert-auth: True

peer-transport-security:
  trusted-ca-file: /etc/pki/tls/certs/ca.pem
  cert-file: /etc/etcd/pki/etcd1.pem
  key-file: /etc/etcd/pki/etcd1-key.pem
  client-cert-auth: True
```

## Set up the configuration environment variables for etcdctl

The command line interface to etcd is called etcdctl. It is configured with
environment variables. As long as you only need to connect to one etcd
cluster, the easiest way to do this is by putting the following file in
/etc/profile.d/etcdctl.sh:

```
export ETCDCTL_CACERT=/etc/pki/tls/certs/ca.pem
export ETCDCTL_CERT=/etc/etcd/pki/etcd1.pem
export ETCDCTL_KEY=/etc/etcd/pki/etcd1-key.pem
export ETCDCTL_ENDPOINTS=etcd1.vagrant:2379,etcd2.vagrant:2379,etcd3.vagrant:2379
```

This will set the correct environment variable for all users who log on
to the system.

This will only work on the etcd servers themselves. If you want to administer
etcd from somewhere else, you must create an additional client certificate for
that system.

## Create the systemd unit file for the etcd service

The etcd service is fairly trivial. Create the following systemd unit file
as /etc/systemd/system/etcd.service:

```
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
User=etcd
Type=notify
ExecStart=/usr/local/sbin/etcd --config-file /etc/etcd/etcd.conf.yml

Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

## Start the cluster and perform a smoke check.

On each node, run

    systemctl start etcd

Then, on any of the nodes, run:

    etcdctl member list


## Backing up data

etcd is backed up with the following command:

    etcdctl snapshot save <filename>.db

In a production cluster, you should run this command in
a cron script, and make sure that the generated file is
saved along with your other backups.

## Other options

etcd can also be set up as a set of pods within Kubernetes itself, as a static
pod. This setup is beyond the scope of this document.

Next: [The front end server/load balancer](./front.md)

