---
title: "Bare Metal to Kubernetes-as-a-Service - Part 2"
excerpt: "Using JUJU and MAAS to deploy Openstack."
layout: single
toc: true
categories:
  - Blog
tags:
  - IaaS
  - Multi-Cloud
  - MaaS
  - Juju
  - OpenStack
  - Kubernetes
---
# Introduction
In [Part 1]({% post_url 2019-08-05-bare-metal-to-kubernetes-part-1 %}) of this series I demonstrated the installation and
configuration of [MAAS].  In this post I'm going to show how to use Canonical's [Juju] to deploy an [Openstack] cloud on top
of hosts managed by MAAS.

>Juju is an open source application modelling tool that allows you to deploy, configure, scale and operate cloud 
>infrastructures quickly and efficiently on public clouds such as AWS, GCE, and Azure along with private ones such as 
>MAAS, OpenStack, and VSphere.

[Juju]:https://jaas.ai/
[MAAS]:https://maas.io/
[Openstack]:https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/index.html

# Prerequisites
If you've been following along from the previous post you should have a working MAAS server with at least one physical 
machine in its default resource pool.  Refer to the [prerequisites] defined in Part 1 for a complete list of requirements.  

Before continuing you'll need to add at least two more machines to MAAS.  I've added an additional physical machine as 
well as a virtual machine composed from a [KVM Pod] on which I'll install my Juju controller.
![MAAS Inventory](/assets/images/20190827/maas_inventory.png "MAAS Inventory")

[KVM Pod]:https://maas.io/docs/kvm-introduction
[prerequisites]:https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-1/#prerequisites

# Install Juju
There are a couple of methods of installing Juju on Debian based Linux [Snapd] and [PPA].  Snap packages are also available for
other Linux distribution.  For MacOS and Windows see the installation documentation.  I'll be using the snap package with
my Linux Mint workstation.

[Snapd]:https://snapcraft.io/docs/installing-snapd
[PPA]:https://launchpad.net/~juju/+archive/ubuntu/stable

To install the snap run the following
```bash
sudo snap install juju --classic
```
## Configure Juju
### Add Clouds
Juju works on top of many different types of [clouds] and has built in support for MAAS.  To see the available clouds run
the following from you workstation console;
```bash
user@host ~$ juju clouds --local

Cloud           Regions  Default          Type        Description
aws                  18  us-east-1        ec2         Amazon Web Services
aws-china             2  cn-north-1       ec2         Amazon China
aws-gov               2  us-gov-west-1    ec2         Amazon (USA Government)
azure                27  centralus        azure       Microsoft Azure
azure-china           2  chinaeast        azure       Microsoft Azure China
cloudsigma           12  dub              cloudsigma  CloudSigma Cloud
google               18  us-east1         gce         Google Cloud Platform
joyent                6  us-east-1        joyent      Joyent Cloud
oracle                4  us-phoenix-1     oci         Oracle Cloud Infrastructure
oracle-classic        5  uscom-central-1  oracle      Oracle Cloud Infrastructure Classic
rackspace             6  dfw              rackspace   Rackspace Cloud
localhost             1  localhost        lxd         LXD Container Hypervisor
```

To add add a new MAAS cloud run the `add-cloud` command and enter the details for your environment.
````bash
user@host ~$ juju add-cloud

Cloud Types
  lxd
  maas
  manual
  openstack
  vsphere

Select cloud type: maas

Enter a name for your maas cloud: maas-01

Enter the API endpoint url: http://10.1.20.5:5240/MAAS/

Cloud "maas-01" successfully added

You will need to add credentials for this cloud (`juju add-credential maas-01`)
before creating a controller (`juju bootstrap maas-01`).
````

[clouds]:https://jaas.ai/docs/clouds

### Add Credentials
As the output shows we'll need to add our MAAS server credentials before Juju can interact with the MAAS API.  To obtain
your MAAS API key, log in to MAAS and click on your username in the top right of the MAAS GUI.
![Obtain Credentials](/assets/images/20190827/obtain_credentials.png "Obtain Credentials")

Copy the API key shown or generate a new key.
![Copy Key](/assets/images/20190827/copy_key.png "Copy Key")

Now run the following and enter the copied API key when prompted.
```bash
stacks@smsmint ~$ juju add-credential maas-01

Enter credential name: maas-01_creds

Using auth-type "oauth1".

Enter maas-oauth: 

Credential "maas-01_creds" added locally for cloud "maas-01".
```

To validate/list the added credentials run;
```bash
stacks@smsmint ~$ juju credentials

Cloud    Credentials
maas-01  maas-01_creds
```

### Add Controllers
juju controllers
juju bootstrap maas-01 juju-01
juju bootstrap maas-01 --to juju-01.maas
juju bootstrap --constraints tags=juju maas-01 juju-01
juju destroy-controller maas-01 --destroy-all-models
juju gui
juju models
juju add-model openstack
juju destroy-model openstack
sudo snap install charm --classic

juju deploy bundle.yaml
watch -c juju status --color

openstack catalog list
openstack endpoint list

juju status openstack-dashboard/0
http://10.1.20.73/horizon

curl http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img | openstack image create --public --min-disk 3 --container-format bare --disk-format qcow2 --property architecture=x86_64 --property hw_disk_bus=virtio --property hw_vif_model=virtio bionic
curl http://cloud-images.ubuntu.com/xenial/current/xenial-server-cloudimg-amd64-disk1.img | openstack image create --public --min-disk 3 --container-format bare --disk-format qcow2 --property architecture=x86_64 --property hw_disk_bus=virtio --property hw_vif_model=virtio xenial
openstack image list

openstack network create --help
openstack network create ext_net --share --external --provider-network-type flat --provider-physical-network physnet1
openstack subnet create ext_subnet --no-dhcp --allocation-pool start=192.168.10.101,end=192.168.10.254 --subnet-range 192.168.10.0/24 --gateway 192.168.10.1 --dns-nameserver 192.168.10.1 --network ext_net
openstack network create int_net
openstack subnet create int_subnet --allocation-pool start=10.0.0.11,end=10.0.0.254 --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 --dns-nameserver 192.168.10.1 --network int_net
openstack router create rtr-01
openstack router set rtr-01 --external-gateway ext_net
openstack router add subnet rtr-01 int_subnet

openstack keypair create --public-key ~/.ssh/do_rsa.pub do_rsa
openstack security group list
PROJECT_ID=$(openstack project show --domain admin_domain admin -f value -c id)
SECGRP_ID=$(openstack security group list --project ${PROJECT_ID} | awk '/default/ {print $2}')
openstack security group rule create ${SECGRP_ID} --protocol any --ethertype IPv6 --ingress
openstack security group rule create ${SECGRP_ID} --protocol any --ethertype IPv4 --ingress
openstack security group rule create b6b7dac6-c616-4bfe-a1d5-285bb499d76c --protocol icmp --remote-ip 0.0.0.0/0
openstack security group rule create b6b7dac6-c616-4bfe-a1d5-285bb499d76c --protocol tcp --remote-ip 0.0.0.0/0 --dst-port 22

openstack flavor create --ram 512 --disk 8 m1.tiny
openstack flavor create --ram 1024 --disk 10 m1.small
openstack flavor create --vcpus 2 --ram 2048 --disk 20 m1.medium
openstack flavor create --vcpus 2 --ram 3584 --disk 20 juju
openstack flavor create --vcpus 4 --ram 4096 --disk 20 m1.large
openstack flavor create --vcpus 12 --ram 16384 --disk 40 m1.xlarge

openstack server create --image bionic --flavor m1.small --key-name do_rsa --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') bionic-test
openstack server create --image xenial --flavor m1.tiny --key-name do_rsa --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') xenial-test

openstack floating ip create ext_net (repeat)
openstack floating ip list
openstack server add floating ip bionic-test 192.168.10.177
openstack server add floating ip xenial-test 192.168.10.187

ssh ubuntu@<new-floating-ip>