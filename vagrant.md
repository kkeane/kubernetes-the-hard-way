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

To allow add VMs to DNS, you have several options:

- You can use the vagrant-dns plugin. This is only supported on MacOS, though.
- You can run a DNS server on your host system. In this case, you must ensure
that your /etc/resolv.conf file points to that DNS server.
- You can ask VirtualBox to inject DNS records into the replies it forwards to
the VMs.

#### To use the vagrant-dns plugin (not tested)

Add the following line to your Vagrantfile

    config.dns.tld = "vagrant"

In each instance stanza in your Vagrantfile, add this line:

    instance.dns.patterns = "ca.vagrant"


#### To inject records into VirtualBox

First, you have to enable the NAT DNS Host Resolver mode and disable the NAT
DNS Proxy mode that conflicts:

```
config.vm.provider "virtualbox" do |vm_config, override|
  vm_config.customize [
    "modifyvm", :id, "--natdnshostresolver1", "on"
  ]
  vm_config.customize [
    "modifyvm", :id, "--natdnsproxy1", "off"
  ]
end
```

Then for each VM, add two entries such as this:

```
vm_config.customize [
  "setextradata", :id, "VBoxInternal/Devices/e1000/0/LUN#0/AttachedDriver/Config/HostResolverMappings/ca/HostIP", "172.28.128.254"
]
vm_config.customize [
  "setextradata", :id, "VBoxInternal/Devices/e1000/0/LUN#0/AttachedDriver/Config/HostResolverMappings/ca/HostNamePattern", "ca.vagrant"
]

```

If you are using a different NIC type than e1000, you may need to adjust the
string. To find out what the correct string is, create a virtual machine with
vagrant up, and then look at the VirtualBox log file (probably in
~/VirtualBox VMs/<machinename>/Logs)

