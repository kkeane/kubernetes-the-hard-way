# Vagrant Notes

I do not actually have physical systems available, and instead use virtual
machines on Vagrant and VirtualBox, on RedHat EL 7. There are a few pitfalls
with such a setup.

- The first, and default NIC in each VM will always have the same IP address
(10.0.2.15). This is not usable for Kubernetes. You must create a second NIC,
and then in a number of places explicitly tell Kubernetes to use this NIC
instead of the default one.

- You should assign static IP addresses to each virtual machine, don't use
DHCP.

- The FQDNs of all VMs must be resolvable through DNS. It is not enough to
create a /etc/hosts file as that is not visible inside containers. If you
are running on MacOS, the vagrant-dns plugin may prove useful here.
Unfortunately, it is not supported (and only partially functional) on Linux.

## Vagrantfile snippets

The complete Vagrantfile is too specific to our environment, but some snippets
should prove generally useful.

### General VM setup

The following sets up the CA virtual machine. Repeat for each of the other
virtual machines, making adjustments as needed.

```
  instance.vm.synced_folder ".", "/vagrant", type: "virtualbox"
  instance.vm.network "private_network", ip: "172.28.128.254"
  instance.vbguest.auto_update = false
  instance.vm.host_name = "ca.vagrant"
  instance.vm.box = "centos/7"
  instance.vm.provider "virtualbox" do |v|
    v.memory = 1024
    v.cpus = 1
  end
```

### To make the VMs resolvable through DNS

To allow add VMs to DNS, we use the the vagrant landrush plugin.

#### To use the landrush plugin

Install the Vagrant plugin with this command:

    vagrant plugin install landrush

If you get a compilation error

    g++: error: unrecognized command line option ‘-Wimplicit-fallthrough=0’

Then your version of gcc is too old. On RedHat-style operating systems version
7, you can install a newer version from the SCL and activate it with

    scl enable devtoolset-7 bash

Details are beyond the scope of this document.

The landrush plugin will require sudo permissions during vagrant runs to
configure dnsmasq and to start it.

More detailed instructions about the landrush plugin are at
https://github.com/vagrant-landrush/landrush

In your vagrant file, you must add this line:

    config.landrush.enabled = true

If you are using an internal corporate DNS server, you must also tell landrush
which DNS server to query next (you should find it in your resolv.conf file):

    config.landrush.upstream '10.1.2.3'

Finally, make sure that landrush knows the TLD that all your VMs are in:

  config.landrush.tld = [ 'vagrant' ]

Next: [Public Key Infrastructure](./pki.md)

