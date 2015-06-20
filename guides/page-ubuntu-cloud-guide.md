---
title: guide.Ubuntu Cloud Guide
layout: page
permalink: /page-ubuntu-cloud-guide/
on_front: true
description: "
A guide based on Ubuntus original 'Install Ubuntu OpenStack'
guide (http://www.ubuntu.com/download/cloud/install-ubuntu-openstack) which I
found a bit lacking in detail.
"
---

Installing The Canonical Distribution of Ubuntu OpenStack - Extended
====================================================================

Introduction (0)
---------------
Canonical provides a short guide on how to use their tools (MAAS, Juju,
Landscape) to quickly set up a fully functional OpenStack [UbuntuOpenStack][].
While following the guide to do exactly that, I noticed that it lacks critical
details in certain areas.

In this guide I want to build upon the one provided by Canonical but add some
more details and explanations where necessary. The section numbers here
directly map to the ones used by Canoncial (so section 1 here refers to step 1
in the official guide).

My goal for using the original guide was to create an OpenStack demo setup on
my laptop as quickly as possible and by running everything in KVM VMs. This
guide here will describe on how to do exactly that. It will address what
resources you absolutely need, how to structure your virtual networks and how
to configure your VMs so everything works as intended. And what to consider
when configuring the tools described in the original guide.

Preparation (1)
---------------

### 1.1 Hardware requirements

Before you start the setup you have to consider the requirements regarding the
host system you are going to work on. The absolute minimum requirements are as
follows:

  * memory: 16 GB (better: 32 GB)
  * disk: 150 GB, SSD
  * operating system with [NestedKVM] enabled

If you have less RAM your system will either swap a lot or you will not be able
to size your cattle VMs (more on that later) in a way that would allow you to
run even the smalles possible instance.

The SSD requirement is a special case for the "one host, multiple VMs"
situation. The process of installing OpenStack as described here creates a lot
of I/O load on the host. After all, there will be 7 active VMs running on your
host at least part of the time. If your disk can not handle that, you risk
timeouts which can lead to random problems / failures during the installation.

### 1.2 Network setup

Contrary to the original guide, the easiest network setup to use involves 3
networks:

  * 1x routed / NAT network with internet access (external)
  * 1x hostonly, no dhcp network, at least 60 IPs (management)
  * 1x hostonly, no dhcp network (public)

The "external" network will only be used on one machine and is the internet
connection for the whole setup. It is necessary because some steps (for
example downloading the images for MAAS) require an internet connection.

The "management" network is used to deploy and access the VMs involved. DHCP
has to be disabled because it will be provided by MAAS later on.

The "public" network will be used exclusively for floating IPs. DHCP is not
needed for that and cloud potentialy cause problems (e.g. handing out IP
addresses for the floating ip range). 

The original guide specifies that only one network is needed, with the floating
IP ranges being part of it. It also specifies that at least two machines need
2 network interfaces, without specifiying to which network the second interface
is connected. In this guide, two networks ("management" + "public") are used
(lets ignore the "external" one for the moment), with the "public" one being
connected to the second network interface.

The specific IP networks used in this guide are as follows:

  * external: 192.168.122.0/24
  * management: 172.31.0.0/16
  * public: 10.34.0.0/16

The gateway for each network is always the host machine, using the first IP of
the network: 192.168.122.1, 172.31.0.1, 10.34.0.1

### 1.3 VM setup

The original guide specifies that you need 7 systems, each with two disks and
two with two network interfaces. Based on that, this guide uses the following
VM setup:

  * 1x deploy VM
    * CPU: 4 cores, copy host CPU configuration
    * networks: external, management
    * disks: 1x 32 GB, virtio, nocache
    * memory: 2048 MB
    * OS: Ubuntu Server 15.04, minimal installation
  * 6x [cattle][] VMs
    * CPU: 2 cores, copy host CPU configuration
    * networks: manmagement, public
    * disks: 1x 32 GB, SATA, nocache ; 1x 10 GB, SATA, nocache
    * memory: 1535 MB
    * PXE boot from management NIC (primary boot method)
    * OS type: most recent Ubuntu available to the KVM installation

This setup massively overcommits the available CPU cores of a normal laptop
(which is ok) but takes care that no memory overcommitment happens. Please
make absolutely sure that you choose "SATA" as the disk type for the cattle
VMs as MAAS is not able to detect virtio disks.

The deploy VM is the only system that has to be set up manually and in the end
will host the MAAS server which manages the "management" network with DHCP and
DNS. It is also the stagin platform for all the automation tools involved.

The cattle systems are just a bunch of identical VMs. The automation tools
(especially Juju) will pick machines at random to deploy necessary services.

With these VMs in place and the networks set up as described, we can go on the
the next step: setting up the deployment VM.

Base setup of the deployment VM (2)
-----------------------------------

The base setup of the deployment VM only consists of two simple steps: setting
up the network and adding all the necessary repositories.

### 2.1 Network setup

As described above, the deployment VM needs two network connections, the
"external" one and the "management one". I am assuming here that "external" is
eth0 and "management" is eth1 (if that is not the case, use whatever interface
names apply). I also assume that your "external" interface is already
configured to use dhcp and */etc/network/interfaces* contains the following
lines for that

~~~
auto eth0
iface eth0 inet dhcp
~~~

What remains to be done is configuring the "management" interface and setting
up masquerading for the "external" interface. As described in the previous
section, our "management" network is 172.31.0.0/16 with 172.31.0.1 being the ip
of the host system. The "management" interface of the deploy VM will therefore
get the ip 172.31.0.2:

~~~bash
sudo cat << EOF > /etc/network/interfaces.d/management
auto eth1
iface eth1 inet static
  address 172.31.0.2
  netmask 255.255.0.0
EOF
ifup eth1
~~~

To allow systems connected to the "management" network to reach the internet,
we need to configure masquerading for the "external" interface. Just extend the
entries for the "external" network in */etc/network/interfaces* as follows:

~~~
auto eth0
iface eth0 inet dhcp
  pre-up iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
~~~

This creates a masquerading rule in iptables for eth0 each time the interface
comes online. Restart the interface and enable packet forwarding:

~~~bash
sudo ifdown eth0
sudo ifup eth0
sudo echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf
sudo sysctl -p
~~~

### 2.2 Enable necessary repositories

As described in the original guide, enable all the package repositories
containing MAAS, juju and the openstack tools:

~~~bash
sudo add-apt-repository ppa:juju/stable
sudo add-apt-repository ppa:maas-maintainers/stable
sudo add-apt-repository ppa:cloud-installer/stable
sudo apt update
~~~

MAAS setup (3)
--------------

### 3.1 Preparation
Start by installing the required packages on the deployment VM.

~~~bash
sudo apt install maas libvirt-bin
~~~

By default MAAS uses the interface with the default gateway as its main
interface (for example for URL generation). In our case this is not what we
want, as the interface with the default gateway is the "external" one but we
want MAAS to provide its services on the "management" interface. To set the
correct ip, execute the following commands:

~~~bash
sudo dpkg-reconfigure maas-cluster-controller
sudo dpkg-reconfigure maas-region-controller
~~~

Just replace the current ip with the one of the "management" interface
(172.31.0.2).

MAAS needs to be able to control (and ideally read) the powerstate of all
systems it manages. As our setup is completely virtualized, MAAS needs to be
able to access our virtualization host (e.g. the laptop everything is running
on). To achieve that "libvirt-bin" was installed alongside MAAS, but we also
need to create an SSH keypair to be used by libvirt to access the KVM host.
MAAS is executed as user "maas" so the keypair needs to be executed by this
user.

Before we do that, enable SSH access on the host system and enable at least
`PermitRootLogin without-password` in */etc/ssh/sshd_config*. For simplicity
reasons this guide uses the root user for managing libvirt on the host.

Create the SSH keypair on the deployment VM:

~~~bash
sudo su -s /bin/bash - maas
  ssh-keygen
  # this "imports" the ssh host key of the KVM host
  ssh root@172.31.0.1
~~~

Copy the contents of */var/lib/maas/.ssh/id_rsa.pub* to
*/root/.ssh/authorized_keys* on the host system. Make sure the SSH connection
from the deployment system to the host system works:

~~~bash
sudo su -s /bin/bash - maas -c "ssh root@172.31.0.1"
~~~

### 3.2 MAAS base setup

Open the MAAS URL http://172.31.0.2/MAAS and follow the instructions displayed
there:

~~~bash
sudo maas-region-admin createadmin
~~~

Reload the website and log in as the newly created user (I will assume its name
is "admin"). Like in the original guide, perform the following steps

  * in the user settings, import your personal SSH key (not the one you created
    earlier in this guide)
  * in the "images" area, import the Ubuntu 14.04. image
  * in the general MAAS settings, set "Upstream DNS used to resolve domains not
    managed by this MAAS" to an upstream DNS server (for example the google DNS
    8.8.8.8)

The upstream DNS server is important because systems connected to the
"management" network will use the DNS server managed by MAAS but they also need
to be able to resolve hostnames outside of the "management" network.


### 3.3 MAAS network setup

MAAS supports two types of networks:

  * networks that can be assigned to VMs to model how many networks are
    connected to a VM (configured in the "networks" area of MAAS)
  * networks that are managed by MAAS (configured in the "clusters" area of
    MAAS)

The 3 networks we have ("external", "management", "public") belong to these as
follows:

  * "external" is neither managed by MAAS nor will it be assigned to VMs
  * "management" will be managed by MAAS and will be assigned to VMs
  * "public" will not be managed by MAAS but it will be assigned to VMs

By default MAAS already imported the two networks of the deployment VM that
were active during the initial MAAS installation. They are called "maas-eth0"
and "maas-eth1". Open the "networks" area of MAAS and delete the "maas-eth0"
network. It is of no concern to MAAS. Open the "maas-eth1" network and
configure it as follows:

  * name: management
  * description: Management and deploy network
  * ip: 172.31.0.0
  * netmask: 255.255.0.0
  * vlan tag: stays empty
  * default gateway: 172.31.0.2
  * dns servers: 172.31.0.2
  * connected network interface cards: will be filled later

Go back to the "networks" area and add a new network. Configure it as follows:

  * name: public
  * description: Public network for OpenStack floating IPs
  * ip: 10.34.0.0
  * netmask: 255.255.0.0
  * vlan tag: stays empty
  * default gateway: 10.34.0.1
  * dns servers: 172.31.0.2
  * connected network interface cards: will be filled later

### 3.4 MAAS cluster setup

Open the "clusters" area and select the only available cluster "Cluster
master". From the list of interfaces, delete eth0. Again, the "external"
network is of no concern to MAAS.

Select the "eth1" interface and configure it as follows:

  * name: management
  * interface: eth1
  * management: DHCP and DNS
  * ip: 172.31.0.2
  * subnet mask: 255.255.0.0
  * broadcast ip: 172.31.255.255
  * router ip: 172.31.0.2
  * DHCP dynamic IP range low value: 172.31.0.50
  * DHCP dynamic IP range high value: 172.31.0.255
  * static IP range low value: 172.31.1.1
  * static IP range high value: 172.31.1.255

Make sure the static and dynamic ip ranges contain at least 30 ips each.

This configuration ensures that all systems in the "management" network will
use the deployment VM as default gateway and DNS server.

### 3.5 DNS setup

To make everything work together, it is necessary that the deployment VM itself
uses the MAAS managed DNS server as it needs to be able to resolve the
automatically generated hostnames for the cattle VMs. Execute the following
commands to set the system nameserver to the MAAS one:

~~~bash
sudo echo "  dns-nameservers 127.0.0.1" >> /etc/network/interfaces.d/management
sudo ifdown eth1
sudo ifup eth1
~~~

The interface restart should probably be done from a VM console.

Check that the maas-dhcpd provides the right ip as DNS server to the clients:

~~~bash
grep domain-name-servers /etc/maas/dhcpd.conf
~~~

If that is not the case (which was the situation everytime I tried that),
change the ip manually to the ip of the deployment VM (172.31.0.2) and restart
the maas-dhcpd:

~~~bash
sudo systemctl restart maas-dhcpd
~~~

### 3.6 Commissioning of the cattle VMs

To make the cattle VMs known to MAAS, just power them on. This will boot them
into the Ubuntu cloud image that has been downloaded in step 3.2 and register
them to MAAS. After that you should see a list of nodes in MAAS, all with
random names.

Set the "Power type" for each node (select the node name -> "Edit Node") to
"Virsh (virtual systems)" and set the "Power parameters" as follows:

  * Power address: "qemu+ssh://root@172.31.0.1/system"
  * Power ID: name of the VM on your VM host
  * Power password: leave empty

Getting the correct power ID is a little bit tricky as the node names in MAAS
provide no indication as to which VM they are referring. The easiest way is to
make note of the MAC addresses listed on the "Edit node" screen and then check
to which VM the MAC addresses belong.

After saving the node settings, the "Check power state" action should return
something like "Success: node is off". If that is not the case, make sure that
the "Power ID" really points to the correct VM name and that the user "maas"
has passwordless ssh access to the root user of your VM host.

At this point, all nodes should be listed as off (black dot in front of the
node name) and in the "new" state in the "Nodes" overview in MAAS.

To complete the commissioning process, select all nodes and perform the
"Commission selected nodes" Bulk action. This should automatically start up all
VMs and perform the necessary commissioning steps. After that all nodes should
have the status set to "Ready".

Ubunut Landscape setup (4)
--------------------------

With MAAS all set up and all availabled systems commissioned and ready to use,
the next step is to set up Juju and Landscape. Juju is the actual "brains" that
will create the OpenStack installation and Landscape is a webgui that allows
you to easily select which components you want to have in your OpenStack
installation.

On the deployment system, install the necessary packages and start the
openstack-install tool

~~~bash
sudo apt install openstack juju-core
sudo openstack-install
~~~

The tool will ask you a few questions about how it should install everything,
answer them as follows:

  * OpenStack Password: the passwqord for the Landscape Admin user
  * Install Type: select "Landscape OpenStack Autopilot"
  * Admin Email: Login name for the Landscape Admin user (can be anything, not
    used to send mails)
  * Admin Name: name of the Admin user (can be anything, not used during login)
  * MAAS Server IP: 172.31.0.2 (management IP of the deploy VM)
  * MAAS API Key: can be found in the admin user settings in MAAS

After entering all details and continuing, the tool should tell you how many
systems are available for installation. If everything went according to plan, 6
systems should be available. Start the installation. Depending on your system
this process can take 20 minutes or longer.

At the end of the process you will be given the URL to the Landscape server.
Open the URL and log in.

OpenStack Installation (5-11)
-----------------------------

To install the actual OpenStack just follow steps 5 to 11 of the original
guide. There are only to things to consider when it comes to step 8:

When configuring the network to be used by OpenStack (after selecting "Open
vSwitch), select the "public" network initially configured in MAAS as the
public network. Sert the start of the floating IP range to 10.34.0.2 as
10.34.0.1 is the VM host itself so it can not be used as floating IP.

You have to select CEPH as Object Storage because SWIFT requires more systems
than are available in the setup described here. If you want to go with SWIFT,
you need two additional cattle VMs.

Before starting the installation take into consideration that this process
probably will take more than an hour to complete.

Troubleshooting
---------------

Although the whole process is highly automated, it is still a very complex
process so here are tips for troubleshooting. 

**Problem:** openstack-install returns "Previous installation detected." and
will not start

**Solution:** delete ~/.cloud-install/ and kill all juju processes (sudo killall
juju)

**Problem:** openstack-install seems to hang. The timer still counts up but
nothing else happens and the VMs seem to be inactive.

**Solution:** Debug juju container
On the deploy VM:

~~~bash
export JUJU_HOME=/home/<user>/.cloud-install/juju
juju status | less
~~~

Look for container with the state other than "running". To connect to a
container, use

~~~bash
juju ssh <container>
~~~

where <container> most likely is "landscape/0". Common causes for a problem
like this are a wrong DNS configuration (check /etc/resolv.conf in an affected
container), not enough disk space or not enough free leases in the DHCP pool
(check the dhcp log on the deploy VM).

**Problem:** the OpenStack installation performed by Landscape seems to hang

**Solution:** Debug juju
This one is very similar to the previous problem but involves a few more
steps.

On the deploy VM:

~~~bash
export JUJU_HOME=/home/<user>/.cloud-install/juju
juju ssh landscape/0
sudo su -s /bin/bash - landscape
export JUJU_HOME=/var/lib/landscape/juju-homes/1 (can also be a different number)
juju status
~~~ 

Look for container with the state other than "running". To connect to a
container, use

~~~bash
juju ssh <container>
~~~

Common causes for a problem like this are a wrong DNS configuration (check
/etc/resolv.conf in an affected container), not enough disk space or not
enough free leases in the DHCP pool (check dhcp log on the deploy VM) or some
network timeout while installing packages.

The best option to solve these problems is to abort the installation, fix the
root cause and start the installation in landscape again.

The "timeout while installing packages" seems to happen sometimes for unknown
reasons, just abort the installation in Landscape and start it again.

  [UbuntuOpenStack]: http://www.ubuntu.com/download/cloud/install-ubuntu-openstack
  [NestedKVM]: http://docs.openstack.org/developer/devstack/guides/devstack-with-nested-kvm.html
  [cattle]: http://jturgasen.github.io/2014/10/25/pets-vs-cattle/
