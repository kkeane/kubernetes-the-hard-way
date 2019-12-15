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

## Certificates needed

TODO

Next: [The worker nodes](./worker.md)
