---
title: "Bare Metal to Kubernetes-as-a-Service - Part 1"
excerpt: "Using MAAS (metal-as-a-service) to build self-service private clouds."
layout: single
toc: true
categories:
  - Blog
tags:
  - IaaS
  - Multi-Cloud
  - MAAS
  - Juju
  - OpenStack
  - Kubernetes
---
# Introduction

>“You can think of Kubernetes as a platform for application patterns. The patterns make your application easy to deploy, 
>easy to run, and easy to keep running.” <cite>[Janet Kuo][1]</cite>

The “ease” with which Google Engineer Janet Kuo speaks has led to the rapid adoption of Kubernetes by organizations of 
all sizes and across all industries.  However, “easy” is relative.  For all of the gains made by Kubernetes, as it relates 
to managing applications, there is a general consensus that deploying and managing Kubernetes itself is hard.  To address 
this problem, all of the major public cloud companies (Google [GKE], Amazon [EKS], Microsoft [AKS], etc.) are now offering 
Kubernetes-as-a-Service.  Similar to Infrastructure-as-a-Service ([IaaS]) these offerings promise to abstract the functionality 
of Kubernetes from the underlying physical resources, tooling and expertise needed to support it.  If your organization 
is leveraging public cloud services then problem solved.  But what about organizations in search of private cloud Kubernetes 
offerings?  Although there are a growing number of options available in this space (OpenShift, Rancher, PKS, Platform9 etc.) 
the degree to which they make Kubernetes “easy” is still relative.

## Objective
To address some of the challenges, to organizations looking to adopt an on-premises solution, I wanted to demonstrate how 
you can build your own Kubernetes-as-a-Service offering using readily available open source solutions. I'm going to cover 
this process in a series of posts each one building upon the previous.  The final solution will consist of a private cloud, 
built with [MAAS] and [Openstack], on which self-service Kubernetes clusters can be deployed.

This first post will cover the deployment of MAAS or Metal-as-a-Service as the base building block of this on-premises 
solution.

# Prerequisites
In order to replicate the configurations that I'm going to demonstrate, the following minimum requirements must be met;
- 1 x Physical host or VM to run MAAS server. (see https://maas.io/docs/maas-requirements)
- 1 x Physical host or VM to run the [Juju] controller. (We'll use this to deploy Openstack on MAAS)
- 2 x Physical hosts with an [IPMI] capable [BMC]. (Minimum Quad Core with 32G RAM recommended)
- 1 x Switch capable of supporting vlans.
- 1 x Router/Firewall to provide inter-vlan routing and Internet access.

## Lab Setup
My lab environment is representative of a typical [brownfield] deployment so disregard the randomness of names, interfaces, 
vlans etc.  The goal is not to confuse but to demonstrate a more complex network configuration than is found in the MAAS 
documentation.

### Physical Configuration
On each of the physical servers that will be managed by MAAS, I'm using a combination of dedicated links, tagged and 
untagged vlan interfaces in order to demonstrate some of the configuration options available in MAAS.  Working with the 
MAAS network model is not very intuitive and requires some trial and error in order to support more complex configurations.  The following lists each interface and its function
in my lab environment.
- eno1 - unused
- eno2 - unused
- eno3 - unused
- eno4 - dedicated for PXE boot (enabled via server bios)
- enp4s0 - 10G trunk link used for management, nfs and Openstack virtual networks
- mgmt0 - iDrac interface used for IPMI power management

![alt text](/assets/images/20190805/physical.png "Lab Setup")

### Logical Configuration
I'm using the following vlans in my lab to represent a minimum set of logical networks one might need to deploy a self
hosted data center.  It is in no way an exhaustive list or a recommendation for best practice.
- v11 - IPMI for out of band access to the physical hosts
- v20 - MAAS for PXE and management of deployed hosts
- v173 - NFS for network attached storage
- v193 - IaaS will be used to assign floating IPs to Openstack instances

![alt text](/assets/images/20190805/maas_lab.png "Lab Setup")

>Note: I will not be covering the creation of vlans or tagged and untagged interfaces on the physical switch.
Please consult your switch vendor's documentation for guidance on implementing these configurations.

# MAAS Deployment
## Install MAAS
The installation of MAAS is very simple.  For lab purposes we'll be installing both the [Region Controller] and 
[Rack Controller] service on the same host.  I recommend you start with a clean install of Ubuntu Server 18.04.  For any 
other OS/version please check the MAAS documentation for compatibility.  To get started you can use any of the following 
options;
- Quick Start - https://maas.io/install
- Documentation - https://maas.io/docs/install-from-packages
- Or for the impatient, ssh to the machine you've dedicated to run MAAS and execute the following;

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
Import SSH keys [] (lp:user-id or gh:user-id): <your_Launchpad_or_Github_name>
```

We won't be using the MAAS CLI for now but if you find you have issues running MAAS commands as a non root user you may 
also need to do the following;
```bash
maas-01:~$ maas --help
Traceback (most recent call last):
  File "/usr/lib/python3/dist-packages/maascli/config.py", line 109, in open
    cls.create_database(dbpath)
  File "/usr/lib/python3/dist-packages/maascli/config.py", line 93, in create_database
    os.close(os.open(dbpath, os.O_CREAT | os.O_APPEND, 0o600))
PermissionError: [Errno 13] Permission denied: '/home/<your_user>/.maascli.db'

maas-01:~$ sudo chown <your_user>:<your_user> .maascli.db
```

You should now be able to login to the MAAS UI at http://(your_maas_ip):5240/MAAS/

## Configure MAAS
The first time you log in to MAAS you'll be prompted to verify/change some initial settings. Of these the following two
need to be configured for your environment.

### DNS forwarder
By default this is set to dns servers used by the host where MAAS is running.

![alt text](/assets/images/20190805/dns_forwarder.png "DNS Forwarder")

### SSH Keys
If you did not specify a Github or Launchpad User-ID during initialization you should upload an RSA public key now.

![alt text](/assets/images/20190805/upload_ssh_key.png "Upload SSH Key")

You can make other changes as desired but for now we will leave the defaults.  

Now click the **Go to dashboard** button to continue configuring MASS.
![alt text](/assets/images/20190805/go_to_dashboard.png "Go to Dashboard")

Next you will be taken to the **Machines** section of the dashboard where you will be presented with a warning that DHCP 
is not enabled.
![alt text](/assets/images/20190805/dhcp_warning.png "DHCP Warning")

The next step in deploying MAAS is to set up the DHCP configuration that will be used to PXE boot hosts.  Before I cover 
these steps it's important to review some of MAAS' [Concepts and Terms] and the different types of DHCP scopes that can 
be configured.  I found the documentation to be a little confusing as it pertains to the setup and configuration of DHCP 
so hopefully this summary will be of use to those deploying MAAS for the first time.
  
### MAAS Concepts and Terms
For complex, highly available deployments, MAAS supports concepts similar to those found in public clouds such as _Regions_
and _Availability Zones_.  For the purposes of this post we will not be focused on those for now.  If you would like 
more information on configuring these features please read the [MAAS Documentation].

#### Fabric
Is synonymous with a Layer 2 Fabric.  It can be represented by a single switch or cluster of connected switches.
It is assumed that vlan IDs will not be duplicated within a single Fabric.

#### Spaces
Are groupings of logical networks that serve a similar purpose and that may or may not be a part of the same Fabric, Zone 
or Region.  An example use of Spaces would be to restrict the deployment of hosts to specific security zones. We won't be 
configuring any spaces for now but I wanted to mention their purpose as its a concept unique to MAAS.

![alt text](/assets/images/20190805/maas_architecture.png "MAAS Networking Concepts")

#### Machines
A machine is a host that can be deployed by MAAS. This can be a physical host or a virtual machine belonging to a [KVM] 
host or [POD].

#### DHCP and Reserved Ranges
This to me was one of the harder configuration concepts to work with in MAAS.  After deploying a
few times and trying different options I was finally able to make sense of the documentation.  The following is a summary
of the concepts as I understand them.

- **DHCP** - At least one [Reserved Dynamic Range] is needed by MAAS for the [Enlistment] and [Commissioning] process 
used to discover and manage machines.  The easiest and recommended setup is to enable DHCP on the untagged vlan in which 
the MAAS server resides.
- **Managed Subnet** - By default all subnets added to MAAS are considered _Managed_ regardless of whether you choose
to enable MAAS managed DHCP for them.
    - **Reserved Ranges** - in a _Managed_ subnet will never be utilized by MAAS.  However, any IP in the subnet that is 
    outside of one of the reserved ranges can be statically assigned to hosts when they are deployed.  Reserved ranges in 
    this case are to be used to exclude the assignment of IPs that may be in use by routers, switches and other infrastructure 
    devices within the subnet.
    - **Reserved Dynamic Ranges** - in a _Managed_ subnet can be used for _Enlistment_ and _Commissioning_ as previously 
    mentioned but can also be used to assign DHCP addresses to hosts during deployment.  This will actually set the deployed 
    hosts' /etc/network/interfaces or [Netplan] configuration to use DHCP.
- **Unmanaged Subnet** - You must explicitly configure a subnet added to MAAS as _Unmanaged_.  This assumes that there 
may be an external DHCP server allocating addresses for this subnet.
    - **Reserved Ranges** - in an _Unmanaged_ Subnet constitute the only addresses that may be statically assigned by MAAS 
    during host deployment.  You must also add this same range to a DHCP exclusion list in any external DHCP server used 
    for this subnet.
    - **Reserved Dynamic Ranges** - can not be configured in an _Unmanaged_ subnet.

Until you feel you've mastered the difference between these concepts it is recommended that you start with a single DHCP
range managed by MAAS for _Enlistment_ and _Commissioning_ purposes and that all other subnets you add be left in the default
configuration of _Managed_.  The following steps will cover the basics of configuring these subnets and reserved ranges.
  
### Configure DHCP
As previously mentioned the first step in configuring the DHCP requirements is to enable MAAS managed DHCP on the default
untagged vlan utilized by the MAAS server.  

>This vlan also needs to be configured as untagged, in your physical switch(s), to 
all hosts that will be managed by MAAS.  It is important to note that if the vlan is not configured as untagged to your 
physical hosts the PXE boot process may not work.

Click on the **Subnets** tab in the MAAS GUI navigation, then click the link for the vlan named **untagged**
![alt text](/assets/images/20190805/configure_dhcp_1.png "Configure DHCP")

Click the **Eanble DHCP** button
![alt text](/assets/images/20190805/configure_dhcp_2.png "Configure DHCP")

To configure the first _Reserved Dynamic Range_ enter a start and end IP address for the range that MAAS will use for DHCP.
I've added a comment as a reminder that this range is primarily used for enlistment and commissioning. Click the **Configure DHCP** 
button to apply the settings.
![alt text](/assets/images/20190805/configure_dhcp_3.png "Configure DHCP")

Now under the **Reserved ranges** section click the drop down menu to the right and select **Reserve range**.  
![alt text](/assets/images/20190805/configure_dhcp_4.png "Configure DHCP")

Since this subnet is a _Managed_ subnet this range of IPs will never be assigned (statically or dynamically) by MAAS.  
We are going to use this range to block off the first ten IPs of the subnet for use by network infrastructure devices.
Enter the range of IPs and then click the **Reserve** button.
![alt text](/assets/images/20190805/configure_dhcp_5.png "Configure DHCP")

At this point we have configured MAAS to manage the allocation of the default vlan/subnet as follows;
- **10.1.20.1-10.1.20.10** - MAAS will never allocate IPs in this range.
- **10.1.20.11-10.1.20.199** - MAAS may use this range to allocate static IPs to deployed hosts.
- **10.1.20.200-10.1.20.254** - MAAS will use this range for Enlistment and Commissioning and may use it to assign DHCP 
addresses to deployed hosts.

### Add Vlans
Next we'll add the additional vlans to be used in our lab environment.

Click on the **Subnets** tab in the MAAS GUI navigation then click the **Add** dropdown menu to the right and select **VLAN**.
![alt text](/assets/images/20190805/add_vlan_1.png "Add Vlans")

Give the vlan a name and set the vlan-id **VID** to match your physical switch configuration.  Here I am configuring my lab's
NFS storage vlan with an 802.1q tag of 173.  Click the **Add VLAN** button to save the new vlan.
![alt text](/assets/images/20190805/add_vlan_2.png "Add Vlans")

Repeat this process for vlan 193 (v193-iaas) or any other vlans you have in your environment.

### Add Subnets
After we've added vlans we need to configure the subnet or subnets that correspond to each vlan.

Click the **Add** dropdown menu in the top right and select **Subnet**.
![alt text](/assets/images/20190805/add_subnet_1.png "Add Subnets")

At minimum you must configure the [CIDR] address and the **Fabric & Vlan** this subnet is associated with.  You may also 
wish to configure a **Gateway IP** and **DNS Servers** (if not using the MAAS server for DNS resolution).  Below I am 
configuring the subnet for my NFS vlan which does not make use of a default gateway or DNS.
![alt text](/assets/images/20190805/add_subnet_2.png "Add Subnets")

>If your underlying network is configured with multiple subnets (v4 or v6) in a single vlan, you can add multiple subnets for each 
>vlan configured in MAAS.

I only have one additional vlan/subnet to add for my lab environment (as previously mentioned, I'll be using this subnet
with Openstack instances in a subsequent blog post).  This subnet is configured with a default gateway and a DNS server 
external to MAAS.
![alt text](/assets/images/20190805/add_subnet_3.png "Add Subnets")

For each of the subnets I've added I will also create a **Reserved range** to block off the first ten IP addresses for 
use by network infrastructure devices.  This step is optional but recommended.

Click the link for the subnet you want to configure.
![alt text](/assets/images/20190805/configure_subnet_1.png "Configure Subnet")

Now under the **Reserved ranges** section click the drop down menu to the right and select **Reserve range**.  Enter the 
desired range of IP addresses to be excluded from assignment by MAAS and click the **Reserve** button.
![alt text](/assets/images/20190805/configure_subnet_2.png "Configure Subnet")

>Recall that by default subnets are considered Managed by MAAS and all IPs (outside of the configured Reserved Ranges) 
>will be available for MAAS to assign as static IPs to deployed hosts.  Since we did not configure any Dynamic Reserved 
>Ranges for these additional subnets they can not be used for Enlistment, Commissioning or to assign DHCP addresses to 
>deployed hosts.

# Machine Deployment
Now that we have completed the initial configuration of MAAS we can prepare the physical hosts/machines that will be 
managed by MAAS.

## Configure Hosts
### Physical Networking
You should verify your physical host cabling and switch configurations to ensure compatibility with the vlans and subnets
that we have added to MAAS.

### Physical Storage
If you have a hardware RAID controller in any of your hosts it will need to be configured prior to adding the host to MAAS.
If this is not done MAAS may not be able to detect storage disks during Enlistment or Commissioning.

### PXE Boot
At least one interface on your physical hosts should be enabled for PXE boot.  As mentioned before this interface should 
be assigned to an untagged switch port in the vlan in which the MAAS server resides.

#### Enable PXE Boot in BIOS
I'm using older Dell R610s with iDrac6 in my lab and will use one of the on board NICs to PXE boot the servers.
![alt text](/assets/images/20190805/pxe_boot_1.png "Enable PXE Boot")

#### Configure BIOS Boot Order
It is important that the PXE enabled interface be placed higher in the boot order than any physical disks.  MAAS will take 
care of ensuring that the server boots from network or disk as needed.
![alt text](/assets/images/20190805/pxe_boot_2.png "Configure Boot Order")

### IPMI
If your servers have an onboard BMC similar to Dell's [DRAC] or HP's [iLO] you can take full advantage of MAAS' power 
control and zero-touch-provisioning capabilities.
 
#### IPMI Settings
For my Dell servers all I had to do was enable IPMI in the DRAC management interface.
![alt text](/assets/images/20190805/ipmi_settings.png "IPMI Settings")

#### IPMI Testing
In order for MAAS to control the power of your servers with IPMI, MAAS will need to be able to reach the servers BMC 
interface via UDP port 623.  You can test that this is working before adding your servers to MAAS by installing **ipmitool** 
on your MAAS server.
```bash
maas-01:~$ sudo apt install ipmitool
maas-01:~$ ipmitool -I lanplus -H <host_ip> -U <user> -P <password> sdr elist all
Temp             | 01h | ns  |  3.1 | No Reading
Temp             | 02h | ns  |  3.2 | No Reading
Temp             | 05h | ns  | 10.1 | No Reading
Ambient Temp     | 07h | ns  | 10.1 | Disabled
Temp             | 06h | ns  | 10.2 | No Reading
Ambient Temp     | 08h | ns  | 10.2 | Disabled
Ambient Temp     | 0Eh | ns  |  7.1 | No Reading
Planar Temp      | 0Fh | ns  |  7.1 | No Reading
CMOS Battery     | 10h | ns  |  7.1 | No Reading
....
...
..
```
To use ipmitool, you must use a username and password that has already been configured in your BMC and that has permissions 
to query IPMI information over LAN.  By default MAAS will create its own user account during the Enlistment phase.

## Add Hosts to MAAS
MAAS manages physical and virtual machines using what it calls a [node lifecycle].  I'm going to focus mainly on the the 
following phases of this lifecycle for now.
- Enlistment
- Commissioning
- Deployment
- Release

In both the Enlistment and Commissioning phases a machine will undergo the following process;
1. DHCP server is contacted
2. kernel and initrd are received over TFTP
3. machine boots
4. initrd mounts a Squashfs image ephemerally over HTTP
5. cloud-init runs enlistment and built-in commissioning scripts
6. machine shuts down

The key difference between the two phases is Enlistment is primarily used for initial discovery and Commissioning allows
for additional hardware customization and testing.

### Enlistment
Adding a host to MAAS is typically done via a combination of DHCP, TFTP and PXE. This unattended manner of adding a host 
is called _Enlistment_.

>Note: MAAS runs built-in commissioning scripts during the enlistment phase so that when you commission a host, any customised 
commissioning scripts you add will have access to data collected during enlistment.

If everything has been configured correctly the only required step to start the Enlistment process is to power on the machine
you are adding to MAAS.  You can watch the Enlistment process discovering a new host in the following video;
{% include video id="jj1M-YyCgD4" provider="youtube" %}

#### IPMI Setup Verification in MAAS
Once the Enlistment process is complete you can verify that MAAS was able to discover/configure the IPMI information needed 
to manage power settings on your host.

Navigate to **Machines** in the MAAS GUI menu.  Click the automatically assigned host name of the newly discovered host,
(we'll change this later) then navigate to the **Configuration** tab.
![alt text](/assets/images/20190805/ipmi_verification_1.png "IPMI Verification")

>Here you can see that the IP address and power management type have been discovered and that MAAS has configured a username
and password to authenticate with the host's BMC.

#### IPMI Setup Verification on Host
You can also confirm that a **maas** user was automatically created in your hosts BMC management interface.
![alt text](/assets/images/20190805/ipmi_verification_2.png "IPMI Verification")

### Commissioning
Once a host has been added to MAAS via the Enlistment process, the next step is to Commission it.
You have the option of selecting some extra parameters (like whether to leave the host running and accessible via ssh) 
and can perform additional hardware tests during this phase.

Prior to Commissioning a host I prefer to change the automatically assigned hostname.  This is purely optional but if you
would like to change the hostname, click the current hostname in the top left corner and set the name to something more 
identifiable for your environment.
![alt text](/assets/images/20190805/change_name.png "Configure Hostname")

To start the commissioning process, click the **Take Action** button in the top right corner and select **Commission** 
from the drop down list.
![alt text](/assets/images/20190805/commission_1.png "Commission Host")

Now you can specify a few configuration options as well as select specific hardware tests to run.  You can leave the default
settings for now and click the **Commission machine** button.
![alt text](/assets/images/20190805/commission_2.png "Commission Host")

You can watch the Commissioning process run its scripts and
perform hardware tests in the following video;
{% include video id="k-9VHZg_qoo" provider="youtube" %}

>Once a host is commissioned its status will change to Ready.

## Deploy Hosts
During the deployment phase MAAS utilizes a process similar to Commissioning to deploy an operating system with [Curtain](https://curtin.readthedocs.io/en/latest/). 
Network and Storage configurations are also applied as part of this process.
1. DHCP server is contacted
2. kernel and initrd are received over TFTP
3. machine boots
4. initrd mounts a Squashfs image ephemerally over HTTP
5. cloud-init triggers deployment process
    1. curtin installation script is run
    2. Squashfs image (same as above) is placed on disk

### Configure MAAS Network Model
Once a machine is in the Ready state (post-commissioning) it's intended network and storage configuration can be defined 
in MAAS.  Interfaces can be added/removed, attached to a fabric, linked to a subnet, and provided an IP assignment mode.  
This is done using a model that represents the desired network configuration for a deployed host.  This allows MAAS to 
deploy consistent networking across multiple operating systems regardless of whether they employ [Network Manager], 
[systemd-networkd] or [Netplan].

>Refer back to the [Lab Setup](#lab-setup) section of this post for an overview of the configurations we'll create in MAAS.

The network configuration we'll model will use a dedicated 10G interface.  This interface is connected to an 802.1q 
trunk on my physical switch and can be used with any vlan configured in my lab environment.

We'll first have to configure the physical interface **enp4s0** then we can add the vlan sub-interfaces.

Navigate to the **Interfaces** tab of the host, click the **Actions** button to the right of **enp4s0** and select 
**Edit Physical** from the list.
![alt text](/assets/images/20190805/configure_interface_1.png "Configure Interfaces")

Add the interface to **fabric-0**, select the **untagged** vlan and it’s associated subnet then set the IP mode to **Auto 
Assign**. Click the **Save physical** button to save the configuration.
![alt text](/assets/images/20190805/configure_interface_2.png "Configure Interfaces")

>I have configured the [native vlan] on the switch port connected to enp4s0 
>to carry the untagged vlan used by the MAAS server. This allows me to utilize this vlan for host management post 
>deployment.

Now we'll add our tagged NFS vlan to the interface.

Click the **Actions** button to the right of **enp4s0** and select **Add alias or VLAN** from the list.
![alt text](/assets/images/20190805/configure_interface_3.png "Configure Interfaces")

Set the **Type** to **VLAN**, select the tagged NFS vlan and its associated subnet, then set the **IP mode** to **Auto assign**.
Now click the **Add** button to save the configuration.
![alt text](/assets/images/20190805/configure_interface_4.png "Configure Interfaces")

Next add the IaaS vlan to the interface.

Click the **Actions** button to the right of **enp4s0** and select **Add alias or VLAN** from the list.
![alt text](/assets/images/20190805/configure_interface_5.png "Configure Interfaces")

Set the **Type** to **VLAN**, select the tagged IaaS vlan and its associated subnet, then set the **IP mode** to **Unconfigured**.
Now click the **Add** button to save the configuration.
![alt text](/assets/images/20190805/configure_interface_6.png "Configure Interfaces")

>It's important that you not configure an IP on this interface else instances created in Openstack will be able to communicate
>directly with the host.

Now we'll remove the existing configuration from the PXE enabled interface **eno4**.  This will only affect the host post deployment.
If the host is [Released](#release) from deployment the interface can still be used to PXE boot the host for further configuration.

Click the **Actions** button to the right of **eno4** and select **Edit Physical** from the list.  To exclude this interface
form the deployment configuration simply set the **Fabric** to **Disconnected**.  Click the **Save physical** button to 
save the configuration.
![alt text](/assets/images/20190805/configure_interface_7.png "Configure Interfaces")

At this point you should have a network model that looks like the following;

![alt text](/assets/images/20190805/configure_interface_8.png "Configure Interfaces")

### Configure MAAS Storage Model
The last requirement before deploying a host is to configure the storage disks and filesystems.

Navigate to the **Storage** tab of the hosts configuration menu and review any existing configuration that may have been 
discovered when the host was added to MAAS.  If no existing partitions and filesystems exist this tab will only show the
disks that were discovered.

>For simplicity I'm going to keep the default storage layout setting of **Flat**.

Next, click the button under **Actions** to the right of the disk you want to configure and select **Add partition**.
![alt text](/assets/images/20190805/configure_disk_1.png "Configure Disks")

Configure the **Size**, **Filesystem** type and **Mount point** you to want provision.  For lab purposes I'm going to use the simplest 
possible configuration of a single ext4 partition mounted at the root of the filesystem.
![alt text](/assets/images/20190805/configure_disk_2.png "Configure Disks")
Click the **Add partition** button after filling in these settings.

### Deployment
To deploy the host click the **Take action** button in the top right corner and select **Deploy** from the drop down list.
You can watch as the Deployment process provisions the storage, OS and network configurations in the following video;
{% include video id="Z4DYUagjjcA" provider="youtube" %}

### Verify Deployment
Once the Deployment process has completed your host should be left powered on with a newly provisioned 
operating system (Ubuntu 18.04 unless otherwise specified).  If the host was deployed successfully, it's status on the
Machines page of the MAAS GUI should show the version of the operating system that was deployed.
![alt text](/assets/images/20190805/os_deployed.png "Verify Deployment")  

You should also be able to ssh to the host using the default 
username **ubuntu** and the SSH key you specified during the initial installation of the MAAS server.
```bash
host:~$ ssh -i ~/.ssh/id_rsa ubuntu@10.1.20.11
```

Once you log in you should verify that your partitions were created as specified and that the MAAS network model properly deployed your 
intended network configuration.
```bash
ubuntu@metal-03:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev             24G     0   24G   0% /dev
tmpfs           4.8G  1.2M  4.8G   1% /run
/dev/sda1        67G  9.8G   54G  16% /
tmpfs            24G     0   24G   0% /dev/shm
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs            24G     0   24G   0% /sys/fs/cgroup
tmpfs           4.8G     0  4.8G   0% /run/user/1000
```
```yaml
ubuntu@metal-03:~$ cat /etc/netplan/50-cloud-init.yaml 

---
network:
    ethernets:
        enp4s0:
            addresses:
            - 10.1.20.94/24
            gateway4: 10.1.20.1
            match:
                macaddress: 5c:40:9e:39:01:d5
            mtu: 1500
            nameservers:
                addresses:
                - 10.1.20.5
                search:
                - maas
            set-name: enp4s0
    version: 2
    vlans:
        enp4s0.173:
            addresses:
            - 172.16.100.11/24
            id: 173
            link: enp4s0
            mtu: 1500
            nameservers:
                addresses:
                - 10.1.20.5
                search:
                - maas
        enp4s0.193:
            id: 193
            link: enp4s0
            mtu: 1500
```

### Release
Now that I have demonstrated the entire deployment process I'm going to release the host back in to MAAS' pool of available
machines.  In the next post I'm going to demonstrate how to use Canonical's [Juju] to deploy a two node Openstack cloud 
with MAAS.

To release the host we just provisioned, go to the the **Machines** page and click the name of the host.  Next click the 
**Take action** button in the top right corner and select **Release** from the drop down list.
![alt text](/assets/images/20190805/release_machine.png "Release Machine")

>Releasing a host back into the pool of available machines changes a host’s status from ‘Deployed’ to ‘Ready’.
>This process includes the ‘Power off’ action. The user has the opportunity to erase the host’s storage (disks) before confirming
>the action.

# Conclusion
The steps and examples demonstrated in this post can be used to construct an entire data center of bare metal resources.  According 
to the documentation, the single MAAS rack controller we deployed can service as many as 1000 hosts.  A distributed, highly-available 
deployment of MAAS is designed to support multiple data centers across multiple regions.  This level of scalability makes
it possible to rapidly build a self managed cloud of physical resources much like those used to power the services of existing public 
cloud vendors.

[1]:https://www.zdnet.com/article/what-kubernetes-really-is-and-how-orchestration-redefines-the-data-center/

[AKS]:https://azure.microsoft.com/en-us/services/kubernetes-service/
[BMC]:https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface#Baseboard_management_controller
[Brownfield]:https://en.wikipedia.org/wiki/Brownfield_(software_development)
[CIDR]:https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing
[Commissioning]:https://maas.io/docs/commission-nodes
[Concepts and Terms]:https://maas.io/docs/concepts-and-terms
[DRAC]:https://en.wikipedia.org/wiki/Dell_DRAC
[EKS]:https://aws.amazon.com/eks/
[Enlistment]:https://maas.io/docs/add-nodes
[GKE]:https://cloud.google.com/kubernetes-engine/
[IaaS]:https://en.wikipedia.org/wiki/Infrastructure_as_a_service
[iLO]:https://en.wikipedia.org/wiki/HP_Integrated_Lights-Out
[IPMI]:https://en.wikipedia.org/wiki/Intelligent_Platform_Management_Interface
[Juju]:https://jaas.ai/
[KVM]:https://www.linux-kvm.org/page/Main_Page
[MAAS]:https://maas.io/
[MAAS Documentation]:https://maas.io/docs
[native vlan]:https://en.wikipedia.org/wiki/IEEE_802.1Q
[Netplan]:https://netplan.io/
[Network Manager]:https://en.wikipedia.org/wiki/NetworkManager
[node lifecycle]:https://maas.io/how-it-works
[Openstack]:https://www.openstack.org/
[POD]:https://maas.io/docs/manage-composable-machines
[Rack Controller]:https://maas.io/docs/introduction-to-controllers
[Region Controller]:https://maas.io/docs/introduction-to-controllers
[Reserved Dynamic Range]:https://maas.io/docs/ip-ranges
[systemd-networkd]:http://man7.org/linux/man-pages/man8/systemd-networkd.service.8.html