---
title: "Why OpenStack is still relevant - Part 1"
excerpt: "Using MAAS (metal-as-a-service) to build self-service private clouds."
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

Introduction - why bare metal, why Openstack, why Kubernetes, why Kubernetes on Openstack on Bare Metal

### Prerequisites
Servers/Networks etc.

### Deployment
1. Install MaaS
2. Configure Maas
  - Image Sources
  - SSH Keys
  - Upstream DNS/Proxy
  - Subnets
3. Configure Host
  - Networking
  - PXE Boot
  - IPMI
    - apt-get install ipmitool
    - ipmitool -I lanplus -H 192.168.0.10 -U root -P calvin sdr elist all
  - Disks

#### Lab Setup
![alt text](/assets/images/20190805/lab_setup.png "Lab Setup")

#### MaaS Networking Concepts
![alt text](/assets/images/20190805/maas_architecture.png "MaaS Networking Concepts")

#### IPMI Settings
![alt text](/assets/images/20190805/ipmi_settings.png "IPMI Settings")

#### IPMI Setup Verification
![alt text](/assets/images/20190805/ipmi_verification_1.png "IPMI Verification")
![alt text](/assets/images/20190805/ipmi_verification_2.png "IPMI Verification")