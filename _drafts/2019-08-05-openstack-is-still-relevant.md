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

## Prerequisites
Servers/Networks etc.

#### Lab Setup
![alt text](/assets/images/20190805/lab_setup.png "Lab Setup")

## Deployment
### Install MaaS
- Quick Start https://maas.io/install
- Documentation https://maas.io/docs/install-from-packages
- For the impatient, ssh to your MaaS host and execute the following;

```bash
maas-01:~$ sudo apt-add-repository -yu ppa:maas/stable
maas-01:~$ sudo apt update
maas-01:~$ sudo apt install maas
maas-01:~$ sudo maas init
Create first admin account
Username: admin
Password: 
Again: 
Email: admin@localhost
Import SSH keys [] (lp:user-id or gh:user-id): <your_Launchpad_or_Gihub_name>
```

- If you find you have issues running maas commands as a non root user you may also need to do the following;

```bash
maas-01:~$ maas --help
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/maascli/config.py", line 109, in open
    cls.create_database(dbpath)
  File "/usr/lib/python3/dist-packages/maascli/config.py", line 93, in create_database
    os.close(os.open(dbpath, os.O_CREAT | os.O_APPEND, 0o600))
PermissionError: [Errno 13] Permission denied: '/home/<user>/.maascli.db'

maas-01:~$ sudo chown <user>:<user> .maascli.db
```

- You should now be able to login to the MAAS UI at http://<your.maas.ip>:5240/MAAS/

### Configure MaaS
- Image Sources
- SSH Keys
- Upstream DNS/Proxy

#### Subnets
  
##### MaaS Networking Concepts
![alt text](/assets/images/20190805/maas_architecture.png "MaaS Networking Concepts")

- DHCP
    
  A reserved dynamic IP range is needed in order for MAAS-managed DHCP to at least enlist and commission nodes and the creation of such a range is part of the process of enabling DHCP with the web UI.
  
  - Enlisted/Commissioned
  - Deployed
- Auto Assigned/Static
  - Deployed
      
### Configure Host
#### Networking

#### PXE Boot
##### Enable PXE boot on NIC
![alt text](/assets/images/20190805/pxe_boot_1.png "Enable PXE Boot")
##### Configure Boot Order
![alt text](/assets/images/20190805/pxe_boot_2.png "Configure Boot Order")
#### IPMI
##### IPMI Settings
![alt text](/assets/images/20190805/ipmi_settings.png "IPMI Settings")
##### IPMI Testing
```bash
maas-01:~$ sudo apt install ipmitool
maas-01:~$ ipmitool -I lanplus -H <host_ip> -U <user> -P <password> sdr elist all
```

##### IPMI Setup Verification in MaaS
![alt text](/assets/images/20190805/ipmi_verification_1.png "IPMI Verification")
##### IPMI Setup Verification on Host
![alt text](/assets/images/20190805/ipmi_verification_2.png "IPMI Verification")

#### Disks




