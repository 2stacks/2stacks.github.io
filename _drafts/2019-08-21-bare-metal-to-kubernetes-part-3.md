---
title: "Bare Metal to Kubernetes-as-a-Service - Part 3"
excerpt: "Using Rancher to deploy self-service Kubernetes clusters"
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

openstack server create --image bionic --flavor m1.medium --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') --user-data cloud-init rancher
openstack server list
openstack server add floating ip rancher <ip>
openstack floating ip list
 
openstack server delete rancher

#### Security Groups
https://rancher.com/docs/rancher/v2.x/en/installation/references/
openstack security group create --description 'Rancher Kubernetes' rancher
openstack security group rule create rancher  --protocol any --remote-group rancher

#### Connect to Rancher UI
https://192.168.10.205

#### Configure Rancher
- Admin Password
- Server URL

#### Enable Openstack Node Driver

#### Add Openstack Node Template
```bash
user@host:~$ env | grep OS_
OS_REGION_NAME=RegionOne
OS_USER_DOMAIN_NAME=admin_domain
OS_PROJECT_NAME=admin
OS_AUTH_VERSION=3
OS_IDENTITY_API_VERSION=3
OS_PASSWORD=thu1laeph8oNgi5U
OS_AUTH_TYPE=password
OS_AUTH_URL=http://10.1.20.57:5000/v3
OS_USERNAME=admin
OS_PROJECT_DOMAIN_NAME=admin_domain

openstack domain list
openstack project list --domain admin_domain

```
- authUrl - http://10.1.20.89:5000/v3
- domainName - admin_domain
- flavorName - m1.large
- floatingipPool - ext_net
- imageName - bionic
- keypairName - do_rsa
- netName - int_net
- password - thu1laeph8oNgi5U
- privateKeyFile - <do_rsa private key>
- secGroups - default
- sshUser - ubuntu
- tenantid - 67b1cc1758dd4f8084e1474db5a2db18 (Use tenantId, using tenantName will cause docker-machine to use identity v2)
- username - admin

#### Add Cluster
- Type Openstack
- Cluster Name
- Node Name Prefix
- Node Template Openstack
- Node Service (etcd, control plane, worker)
- Openstack Cloud Provider

openstack network list

```yaml
cloud_provider:
  name: openstack
  openstackCloudProvider:
    global: 
      username: admin
      password: thu1laeph8oNgi5U
      auth-url: "http://10.1.20.89:5000/v3"
      tenant-id: 67b1cc1758dd4f8084e1474db5a2db18
      domain-id: 5d7abf55dc474bacb33c90ca3bd9fdd0
    load_balancer:
      use-octavia: true
      subnet-id: 0610f777-afff-4228-850c-5d9d2b06cfaf
      floating-network-id: 6de04c91-70dc-477c-8e7d-0f85496c4996
    route:
      router-id: cdc4e1b1-a92d-47d1-85a2-03c6209d68c5
```

juju destroy-model openstack
juju destroy-controller juju-01 --destroy-all-models
juju remove-cloud maas-01
juju remove-credential maas-01  maas-01_cred