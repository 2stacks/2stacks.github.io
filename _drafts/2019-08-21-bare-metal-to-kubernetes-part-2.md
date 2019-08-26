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

juju clouds
juju controllers
juju add-cloud
juju add-credential maas-01
juju credentials  --show-secrets
juju remove-cloud maas-01
juju bootstrap maas-01 juju-01
juju bootstrap maas-01 --to juju-01.maas
juju bootstrap --constraints tags=juju maas-01 juju-01
juju destroy-controller maas-01 --destroy-all-models
juju gui
juju models
juju add-model openstack
juju destroy-model openstack
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