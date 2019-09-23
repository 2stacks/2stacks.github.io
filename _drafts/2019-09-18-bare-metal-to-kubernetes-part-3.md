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
  - Rancher
---
# Introduction
Why Rancher?

# Prerequisites
- [Part 1]({% post_url 2019-08-05-bare-metal-to-kubernetes-part-1 %})
- [Part 2]({% post_url 2019-09-06-bare-metal-to-kubernetes-part-2 %})

# Install Rancher
[Rancher Docs]:https://rancher.com/docs/rancher/v2.x/en/

## Update cloud-init
[cloud-init]:https://cloudinit.readthedocs.io/en/latest/
Get SSH Public Key
[Part 2](https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-2/#add-ssh-keys)
```bash
openstack keypair show --public-key os_rsa
```
Download user data file
```bash
wget 
```

Edit user data file with your public key

## Create Rancher Server
```bash
openstack server create --image bionic --flavor m1.medium --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') --user-data cloud-init rancher
openstack server list rancher
```

## Add Floating IP
To enable SSH access from external hosts we'll need to assign the instance a floating ip in the external subnet. 
```bash
floating_ip=$(openstack floating ip create -f value -c floating_ip_address ext_net)
openstack server add floating ip rancher ${floating_ip}
```

# Security Groups
[port and protocol]:https://rancher.com/docs/rancher/v2.x/en/installation/requirements/

```bash
project_id=$(openstack project show --domain admin_domain admin -f value -c id)
secgroup_id=$(openstack security group list --project ${project_id} | awk '/default/ {print $2}')
openstack security group rule create $secgroup_id --protocol any --ethertype IPv6 --ingress
openstack security group rule create $secgroup_id --protocol any --ethertype IPv4 --ingress
```

# Connect to Rancher UI
```bash
openstack server show rancher -f value -c addresses
```
`https://<floating_ip>`

![Rancher UI](/assets/images/20190918/rancher_ui.png "Rancher UI")

# Configure Rancher
- Admin Password
- Server URL

![Rancher URL](/assets/images/20190918/rancher_url.png "Rancher URL")

## Enable Openstack Node Driver
Tools -> Drivers -> Node Drivers

![Enable Node Driver](/assets/images/20190918/node_driver_1.png "Enable Node Driver")

![Enable Node Driver](/assets/images/20190918/node_driver_2.png "Enable Node Driver")

## Add Openstack Node Template
User Menu -> Node Templates -> Add Template

![Enable Node Template](/assets/images/20190918/node_template_1.png "Enable Node Template")

![Enable Node Template](/assets/images/20190918/node_template_2.png "Enable Node Template")

### Obtain Template Data
```bash
~$ source openrc
~$ env | grep OS_

OS_USER_DOMAIN_NAME=admin_domain
OS_PASSWORD=Uafe7chee6eigedi
OS_AUTH_URL=http://10.1.20.32:5000/v3
OS_USERNAME=admin
```

Get domain and tenant id
```bash
openstack project show admin -f value -c id
```
- authUrl - http://10.1.20.32:5000/v3
- domainName - admin_domain
- flavorName - m1.large
- floatingipPool - ext_net
- imageName - bionic
- insecure - enable
- keypairName - os_rsa
- netName - int_net
- password - Uafe7chee6eigedi
- privateKeyFile - <os_rsa private key>
- secGroups - default
- sshUser - ubuntu
- tenantid - ca52ba242b104464b15811d4589f836e (Use tenantId, using tenantName will cause docker-machine to use identity v2)
- username - admin

![Enable Node Template](/assets/images/20190918/node_template_3.png "Enable Node Template")

# Add Cluster
Clusters -> Add Cluster

![Add Cluster](/assets/images/20190918/add_cluster_1.png "Add Cluster")

- Type Openstack
- Cluster Name - os1
- Node Name Prefix - os1-all-
- Count - 3
- Node Template Openstack
- Node Service - etcd, control plane, worker
- Openstack Cloud Provider

![Add Cluster](/assets/images/20190918/add_cluster_2.png "Add Cluster")

![Add Cluster](/assets/images/20190918/add_cluster_3.png "Add Cluster")

## Configure the Cloud Provider
Cloud Provider Custom

![Cloud Provider](/assets/images/20190918/cloud_provider_1.png "Cloud Provider")

![Cloud Provider](/assets/images/20190918/cloud_provider_2.png "Cloud Provider")

```bash
OS_PASSWORD=Uafe7chee6eigedi
OS_AUTH_URL=http://10.1.20.32:5000/v3
OS_USERNAME=admin
```

```bash
openstack project show admin -f value -c id
openstack project show admin -f value -c domain_id
openstack subnet show int_subnet -f value -c id
openstack network show ext_net -f value -c id
openstack router show rtr-01 -f value -c id
```

```yaml
cloud_provider:
  name: openstack
  openstackCloudProvider:
    global: 
      username: admin
      password: Uafe7chee6eigedi
      auth-url: "http://10.1.20.32:5000/v3"
      tenant-id: ca52ba242b104464b15811d4589f836e
      domain-id: 3f7614e901ef4adc8e14014ae9a5d8e6
    load_balancer:
      use-octavia: true
      subnet-id: a114a47d-e7d5-4e98-8a5a-26b2a6692017
      floating-network-id: e499e0b6-fa1e-40b0-abcd-3f9299fd1094
    route:
      router-id: 498ce328-6ba2-4a09-b421-71e28028b4fa
```

![Cloud Provider](/assets/images/20190918/cloud_provider_3.png "Cloud Provider")

Kubernetes Openstack [Cloud Provider] documentation or the [Rancher] documentation.

[Cloud Provider]:https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#openstack
[Rancher]:https://rancher.com/docs/rke/latest/en/config-options/cloud-providers/openstack/

## Validate Cluster
Cluster Name -> Nodes

![Validate Cluster](/assets/images/20190918/validate_cluster_1.png "Validate Cluster")

```bash
$ openstack server list
+--------------------------------------+-----------+--------+------------------------------------+--------+-----------+
| ID                                   | Name      | Status | Networks                           | Image  | Flavor    |
+--------------------------------------+-----------+--------+------------------------------------+--------+-----------+
| 6a54ade8-1c93-486e-99d9-183b401131a7 | os1-all-2 | ACTIVE | int_net=10.0.0.127, 192.168.10.229 | bionic | m1.large  |
| be07abea-fab0-4ffd-8098-0e6c4b69a86b | os1-all-1 | ACTIVE | int_net=10.0.0.41, 192.168.10.180  | bionic | m1.large  |
| 8a281b10-2ab9-4573-b1bc-62910622fb46 | os1-all-3 | ACTIVE | int_net=10.0.0.81, 192.168.10.204  | bionic | m1.large  |
| 982c2885-55fb-4a5b-a9ee-8bd4b9d2a771 | rancher   | ACTIVE | int_net=10.0.0.238, 192.168.10.234 | bionic | m1.medium |
+--------------------------------------+-----------+--------+------------------------------------+--------+-----------+
```
> Make sure that the status of all hosts is **ACTIVE** and that each is associated with a floating IP address.

You can also monitor the cluster build process by connecting to the Rancher instance vi SSH.
```bash
ssh -i ~/.ssh/os_rsa ubuntu@<rancher_ip>
```

Get the rancher Container ID.
```bash
ubuntu@rancher:~$ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS                                      NAMES
21792badbb1c        rancher/rancher:v2.2.8   "entrypoint.sh"     19 hours ago        Up 19 hours         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   upbeat_chaplygin
```

Follow logs for the rancher container.
```bash
docker logs -f 21792badbb1c
```

![Validate Cluster](/assets/images/20190918/validate_cluster_2.png "Validate Cluster")

# Deploy Application
Cluster Name -> Default Namespace -> Deploy

![Deploy App](/assets/images/20190918/deploy_app_1.png "Deploy App")

![Deploy App](/assets/images/20190918/deploy_app_2.png "Deploy App")

## Configure Workload
- Name:         webapp
- Workload Type: 3 pods
- Docker Image: [leodotcloud/swiss-army-knife](https://github.com/leodotcloud/swiss-army-knife)

![Deploy App](/assets/images/20190918/deploy_app_3.png "Deploy App")

Add Port

- Container Port: 80
- Protocol: TCP
- As a: L4 Load Balancer
- Listening Port: 80

Environment Variables -> Add Variable

- ALPHABET: A

![Deploy App](/assets/images/20190918/deploy_app_4.png "Deploy App")

# Validate Application

## View Application Status

![Validate App](/assets/images/20190918/validate_app_1.png "Validate App")

![Validate App](/assets/images/20190918/validate_app_2.png "Validate App")

## View Load Balancer Status

![Validate LB](/assets/images/20190918/validate_lb_1.png "Validate LB")

```bash
$ openstack loadbalancer list
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| id                                   | name                             | project_id                       | vip_address | provisioning_status | provider |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| 1301cf7b-9319-4222-a0a0-bcf42fe6d2f0 | ad023b4dbda2e11e98886fa163ea07cc | ca52ba242b104464b15811d4589f836e | 10.0.0.50   | PENDING_CREATE      | amphora  |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
```

```bash
$ openstack loadbalancer list                             
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| id                                   | name                             | project_id                       | vip_address | provisioning_status | provider |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| 1301cf7b-9319-4222-a0a0-bcf42fe6d2f0 | ad023b4dbda2e11e98886fa163ea07cc | ca52ba242b104464b15811d4589f836e | 10.0.0.50   | ACTIVE              | amphora  |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
```

![Validate LB](/assets/images/20190918/validate_lb_2.png "Validate LB")

# Test Application

![Test App](/assets/images/20190918/test_app_1.png "Test App")

![Test App](/assets/images/20190918/test_app_2.png "Test App")

Browser (refresh)

![Test App](/assets/images/20190918/test_app_3.png "Test App")

# Conclusion

