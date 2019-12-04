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

TODO

## Create the etcd configuration file

TODO

## Set up the configuration environment variables for etcdctl

TODO

## Create the systemd unit file for the etcd service

TODO

## Backing up data

TODO

