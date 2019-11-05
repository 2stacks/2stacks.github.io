---
title: "Getting Started with FRRouting"
excerpt: "Installing and Running FRR on Eve-NG"
layout: single
toc: true
categories:
  - Blog
tags:
  - FRRouting
  - Eve-NG
  - Open Source
---
# Introduction
If you've been around networking long enough chances are you have heard of the open source routing software [Zebra] or 
[Quagga].  You may have used them without knowing if you ever worked with Sidewinder firewalls or Brocade's Vyatta.  If 
you :heart: Linux and open source routing you may have heard of the next evolution in open source routing [FRRouting] 
aka Free Range Routing (FRR).  If you haven't;

> FRR is a routing software package that provides TCP/IP based routing services with routing protocols support such as 
>BGP, RIP, OSPF, IS-IS and more.  It uses the Linux kernel's routing stack for packet forwarding.

FRR has been around since 2016 when it was forked from the Quagga project.  It has a growing number of contributors and 
support from various networking vendors such as Cumulus, VMware, Volta, Orange and 6wind with the common goal of creating 
a "world class routing stack."  The FRR project is currently licensed under the [GPLv2+] and is overseen by [The Linux 
Foundation] in an effort to ensure the project's growth and sustainability.

In this post I'm going to demonstrate how to install FRR on the popular network modeling platform [Eve-NG].  This will 
allow you to evaluate FRR and its many features and hopefully inspire you to get more involved with open source 
routing.

# Prerequisites
If you are not familiar with [Eve-NG] or Emulated Virtual Environment (EVE), it is a platform for designing, testing and
training with virtual appliances and network emulation software.  Although geared more towards networking, you can emulate 
almost anything in Eve-NG that can run on the [KVM] hypervisor.  Since FRR is a routing platform Eve-NG provides everything
you need to start creating labs and proof of concepts using FRR.

Eve-NG can run on bare metal or as a virtual appliance on KVM and VMware.  There are a few different [installation] methods
so I'm not going to cover that in this post.  Once you have a running instance of Eve-NG you can follow along
with how to set up FRR for use within Eve-NG.

## Client Tools
Whether you choose to work from a Windows, Mac or Linux OS you'll need the following tools to work with Eve-NG and to install
FRR.
  - VNC
  - Telnet
  - SSH
  
It is highly recommended that you install the Eve-NG [Client Integration] tools for your specific OS.

[installation]:http://www.eve-ng.net/documentation/installation/system-requirement

# Installation
Before we start installing FRR its important to check the FRR [Releases] page.  Here you'll find a list of versions and
their supported features as well as the different installation methods for your chosen base operating system.

If you are interested in testing a specific feature you should also review the [Supported Protocols] section of the FRR
documentation.  Here you'll find the OS and kernel version requirements for all of FRR's supported features.

It's important to note that some of the newest features of FRR require the latest linux kernel versions.  I'm going to 
start with a standard installation of the [Ubuntu Server] version 18.04.3 LTS and demonstrate how to upgrade to the most 
recent kernel versions.

The process I'm going to use is well documented in the Eve-NG [Linux How-To] documentation.  I've only added a few steps
I've found useful for working with FRR.  

## Get Ubuntu Server ISO
Using SSH or the console, log in as `root` to your Eve-NG server and download the latest LTS version of the [Ubuntu Server].

```bash
root@eve-ng:~# wget http://releases.ubuntu.com/18.04.3/ubuntu-18.04.3-live-server-amd64.iso
```

## Create qemu directories and images
From your Eve-NG server create the required directories and image files.  Eve-NG requires the directory be prefixed with
`linux-`.  I'm going to install FRR using the latest available Debian package version 7.1.  I've chosen a base disk size
of 16GB but you can use any size you like so long as it meets the Ubuntu server [minimum requirements].
```bash
cd /opt/unetlab/addons/qemu/
mkdir linux-frr-7.1 && cd ./linux-frr-7.1
cp /root/ubuntu-18.04.3-live-server-amd64.iso cdrom.iso
/opt/qemu/bin/qemu-img create -f qcow2 virtioa.qcow2 16G
```

Before continuing verify the creation of the [qemu] image file.
```bash
root@eve-ng:/opt/unetlab/addons/qemu/linux-frr-7.1# /opt/qemu/bin/qemu-img info virtioa.qcow2  
image: virtioa.qcow2
file format: qcow2
virtual size: 16G (17179869184 bytes)
disk size: 196K
cluster_size: 65536
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 
```

## Create Eve-NG Lab
Now log in to the web interface of your Eve-NG server. 

![Eve-NG Login](/assets/images/20191105/eve_login.png "Eve-NG Login")

From the root of the file manager click the icon to add a new lab.

![Add Lab](/assets/images/20191105/add_lab.png "Add Lab")

Give the lab a name and description similar to `FRR Test`.

![Add Lab](/assets/images/20191105/add_lab_1.png "Add Lab")

Right click anywhere inside the new lab and select **Node** from the new object menu.

![Add Node](/assets/images/20191105/add_node.png "Add Node")

Scroll down the new node list and select **Linux**

![Add Node](/assets/images/20191105/add_node_1.png "Add Node")

If this is the first linux image you've added to Eve-NG you should only see the **linux-frr-7.1** in the list.  Otherwise
choose the FRR image from the image list.

![Add Node](/assets/images/20191105/add_node_2.png "Add Node")

To make the installation faster you can add more CPU and RAM to the node settings.
All other settings can be left at their defaults.  Click **Save** once you've made your changes.

![Add Node](/assets/images/20191105/add_node_3.png "Add Node")

In order to install updates and the FRR packages to the instance, you'll need to connect it to your local network.  Eve-NG
uses **Cloud** networks to bridge virtual instances to interfaces on your Eve-NG server.  This allows you to connect your
lab instances to the same network used by Eve-NG for access to the Internet.

Right click in the open space of your lab and select **Network** from the new object menu.

![Add Network](/assets/images/20191105/add_network.png "Add Network")

Give the network a name and change the **Type** to **Management(Cloud0)**.  

>This will bridge the instance of the Ubuntu
server to the same network used by your Eve-NG server.  If this network is not enabled with DHCP you will have to configure
a static IP on the Ubuntu server during the installation process.

![Add Network](/assets/images/20191105/add_network_1.png "Add Network")

To connect the node to the newly created cloud network, mouse over the node and drag the orange plug icon from the node
to the cloud network icon.

![Connect Node](/assets/images/20191105/connect_node.png "Connect Node")

The **Add Connection** menu will appear and allow you to specify which interface on your Ubuntu server instance you want
to connect to the cloud network.  Unless you added additional interfaces when you added your node you should only see an
option for **e0**.

![Connect Node](/assets/images/20191105/connect_node_1.png "Connect Node")

Click **Save** to complete the connection.

![Connect Node](/assets/images/20191105/connect_node_2.png "Connect Node")

Now right click on the node and select **Start** from the node menu.

![Start Node](/assets/images/20191105/start_node.png "Start Node")

If you've installed the Eve-NG [Client Integration] tools you should now be able to click on the node to connect with [VNC].  If
you did not you'll need to open your VNC client and manually connect to the instance.  You can find the VNC connection URL
by mousing over the node and looking in the bottom left hand corner of your browser window.

![Connect To Node](/assets/images/20191105/connect_to_node.png "Connect To Node")

## Install Ubuntu Server

Once you're connected to the instance with VNC proceed with the installation of the Ubuntu Server OS.

![Install OS](/assets/images/20191105/install_os.png "Install OS")

Accept all of the defaults until you reach the **Filesystem Setup** screen.  Here you should choose the **Manual** option.

![Install OS](/assets/images/20191105/install_os_1.png "Install OS")

Select **/dev/vda** from the list of available devices.

![Install OS](/assets/images/20191105/install_os_2.png "Install OS")

Now select **Add Partition**.

![Install OS](/assets/images/20191105/install_os_3.png "Install OS")

Enter the max size available based on the size of the qemu image you created earlier and select **Create**

![Install OS](/assets/images/20191105/install_os_4.png "Install OS")

>The max size should be just shy of the total disk size as the installation has already reserved space for [Grub].

Now select **Done**.

![Install OS](/assets/images/20191105/install_os_5.png "Install OS")

Select **Continue** to begin the installation.

![Install OS](/assets/images/20191105/install_os_6.png "Install OS")

When you reach the **Profile Setup** menu enter a default username and password to be used to access the FRR instance.

**NOTE: Do not use `frr` as the username as this will break the FRR installation.**

![Install OS](/assets/images/20191105/install_os_7.png "Install OS")

Select the option to enable the OpenSSH server.  You can optionally import SSH keys if you know what you are doing.

![Install OS](/assets/images/20191105/install_os_8.png "Install OS")

Do not select any additional Snap packages for installation, scroll to the bottom of the screen and select **Done**.

![Install OS](/assets/images/20191105/install_os_10.png "Install OS")

Allow the system to fully install and update packages.  Do not select the option to **Cancel update and reboot**.

![Install OS](/assets/images/20191105/install_os_11.png "Install OS")

Once the installation and OS update have completed select the option to **Reboot**.

![Install OS](/assets/images/20191105/install_os_12.png "Install OS")

The reboot will pause and prompt you to remove the installation medium.

![Install OS](/assets/images/20191105/install_os_13.png "Install OS")

At this point you should connect to your Even-NG server and remove the `cdrom.iso` file you created from the Ubuntu Server 
installation ISO.

```bash
cd /opt/unetlab/addons/qemu/linux-frr-7.1
rm -rf cdrom.iso
```

Now from the Eve-NG GUI right click on your node and select **Stop** from the node menu.

![Stop Node](/assets/images/20191105/stop_node.png "Stop Node")

Right click on the node again and select **Start** from the node menu.

>It is important to fully stop and then restart the node to ensure that it does not boot from a cached copy of the 
>installation ISO.

![Restart Node](/assets/images/20191105/restart_node.png "Restart Node")

Now click on the node or manually launch your VNC client as before to connect to the console of the node.

![Log In to Node](/assets/images/20191105/login_to_node.png "Log In to Node")

## Upgrade Linux Kernel
To get the latest 5.0 kernel versions in the LTS release of Ubuntu Server you'll need to set up the [LTS Enablement Stack].
You can read more about it in this [HWE Kernel Install] guide or just follow the steps below.

>By default, Ubuntu LTS releases stay on the same Linux kernel they were released with. The hardware enablement stack 
>(HWE) provides newer kernel and xorg support for existing Ubuntu LTS releases.

Login to the FRR system using the username and password you created during the Ubuntu Server installation.

First verify the current kernel version.
```bash
~$ lsb_release -a
No LSB modules are available.
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.3 LTS
Release:        18.04
Codename:       bionic

~$ uname -a
Linux frr-01 4.15.0-66-generic #75-Ubuntu SMP Tue Oct 1 05:24:09 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

To upgrade to latest kernel version run the following;
```bash
sudo apt-get install --install-recommends linux-generic-hwe-18.04
sudo apt-get update
sudo apt-get dist-upgrade
sudo reboot
```

Now verify that you are on the latest 5.0 kernel version.  If the kernel wasn't updated run the `dist-upgrade` and `reboot` again.
```bash
~$ uname -a
Linux frr-01 5.0.0-32-generic #34~18.04.2-Ubuntu SMP Thu Oct 10 10:36:02 UTC 2019 x86_64 x86_64 x86_64 GNU/Linux
```

## Add Debian Repo and Install packages
Now install FRR from the [FRR Debian Repository](https://deb.frrouting.org/).
```bash
curl -s https://deb.frrouting.org/frr/keys.asc | sudo apt-key add -
FRRVER="frr-stable"
echo deb https://deb.frrouting.org/frr $(lsb_release -s -c) $FRRVER | sudo tee -a /etc/apt/sources.list.d/frr.list
sudo apt update && sudo apt install frr frr-pythontools
```

## Add Users to frrvty Group
To allow access to FRR's [vtysh] configuration utility you must add users to the **frrvty** group.
```bash
sudo usermod -a -G frr,frrvty <your_username>
```

## Enable Daemons
By default only [zebra] and [staticd] are enabled.
```bash
~$ ps -ef | grep frr
root       973     1  0 21:48 ?        00:00:00 /usr/lib/frr/watchfrr -d zebra staticd
frr        999     1  0 21:48 ?        00:00:00 /usr/lib/frr/zebra -d -A 127.0.0.1 -s 90000000
frr       1056     1  0 21:48 ?        00:00:00 /usr/lib/frr/staticd -d -A 127.0.0.1
```

To enable additional daemons edit the `/etc/frr/daemons` file with your preferred text editor.  Change the `no` to `yes` 
for each daemon you want to enable.
```bash
~$ sudo vi /etc/frr/daemons

bgpd=yes
ospfd=yes
ospf6d=yes
ripd=yes
ripngd=yes
isisd=yes
pimd=yes
ldpd=yes
nhrpd=yes
eigrpd=yes
babeld=no
sharpd=yes
pbrd=yes
bfdd=yes
fabricd=yes
```

Restart frr with systemd.
```bash
sudo systemctl restart frr.service
```

Verify that the new daemons have started.
```bash                 
~$ ps -ef | grep frr
root      2149     1  0 00:49 ?        00:00:00 /usr/lib/frr/watchfrr -d zebra bgpd ripd ripngd ospfd ospf6d isisd pimd ldpd nhrpd eigrpd pbrd staticd bfdd fabricd
frr       2208     1  0 00:49 ?        00:00:00 /usr/lib/frr/zebra -d -A 127.0.0.1 -s 90000000
frr       2213     1  0 00:49 ?        00:00:00 /usr/lib/frr/bgpd -d -A 127.0.0.1
frr       2221     1  0 00:49 ?        00:00:00 /usr/lib/frr/ripd -d -A 127.0.0.1
frr       2225     1  0 00:49 ?        00:00:00 /usr/lib/frr/ripngd -d -A ::1
frr       2230     1  0 00:49 ?        00:00:00 /usr/lib/frr/ospfd -d -A 127.0.0.1
frr       2235     1  0 00:49 ?        00:00:00 /usr/lib/frr/ospf6d -d -A ::1
frr       2239     1  0 00:49 ?        00:00:00 /usr/lib/frr/isisd -d -A 127.0.0.1
frr       2243     1  0 00:49 ?        00:00:00 /usr/lib/frr/pimd -d -A 127.0.0.1
frr       2248     1  0 00:49 ?        00:00:00 /usr/lib/frr/ldpd -L
frr       2249     1  0 00:49 ?        00:00:00 /usr/lib/frr/ldpd -E
frr       2250     1  0 00:49 ?        00:00:00 /usr/lib/frr/ldpd -d -A 127.0.0.1
frr       2256     1  0 00:49 ?        00:00:00 /usr/lib/frr/nhrpd -d -A 127.0.0.1
frr       2260     1  0 00:49 ?        00:00:00 /usr/lib/frr/eigrpd -d -A 127.0.0.1
frr       2281     1  0 00:49 ?        00:00:00 /usr/lib/frr/bfdd -d -A 127.0.0.1
frr       2284     1  0 00:49 ?        00:00:00 /usr/lib/frr/pbrd -d -A 127.0.0.1
frr       2287     1  0 00:49 ?        00:00:00 /usr/lib/frr/fabricd -d -A 127.0.0.1
frr       2291     1  0 00:49 ?        00:00:00 /usr/lib/frr/staticd -d -A 127.0.0.1
```

>Keep in mind that each daemon requires an additional [pthread] and memory and BGP requires more than one additional
>thread.  In production is is not unreasonable to have a CPU per enabled daemon.  At minimum BGP would require 2 CPUs and
>~600MB of memory (per table) for a full Internet table of routes.

## Enable Kernel Forwarding
IP forwarding is disabled by default on most linux distributions so in order to perform any routing you must enable ip 
forwarding in the kernel by editing `/etc/systctl.conf` and uncommenting the following lines.
```bash
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

>Some documentation lists the setting to enable IPv4 forwarding as `net.ipv4.conf.all.forwarding=1`.  Verify which setting
>is appropriate for you system.

### Surviving Reboots
At the time of this writing there is a bug in Ubuntu that prevents some sysctl settings from loading after reboot.

One workaround is to create a new file `/etc/rc.local` and add the following contents:
```bash
#!/bin/bash
# /etc/rc.local

# Load kernel variables from /etc/sysctl.d
/etc/init.d/procps restart

exit 0
```

Make sure that the file is executable by running;
```bash
sudo chmod 755 /etc/rc.local
```

## Optional Settings
Additional [sysctl settings and kernel modules] are required for [MPLS] and [VRFs].  Please see the FRR installation
documentation for these settings.

# Initial FRR Configuration
If you want to set some basic defaults in FRR that will be applied to all new instances created in Eve-NG, you should 
configure them prior to committing any changes back to the base qemu image. 

You can configure FRR directly via its config files or interactively using [vty] or [vtysh].

To start an interactive shell using vtysh run the vtysh command.  Keep in mind that the currently logged in user must be 
a member of the frrvty group in linux.
```bash
~$ vtysh

Hello, this is FRRouting (version 7.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

Router#
```

In order to use vty you'll need to configure a vty and enable password.
```bash
~$ vtysh

Hello, this is FRRouting (version 7.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

Router# configure terminal 
Router(config)# password zebra
Router(config)# enable password zebra
Router(config)# exit
Router# write memory 
Note: this version of vtysh never writes vtysh.conf
Building Configuration...
Integrated configuration saved to /etc/frr/frr.conf
[OK]
Router# exit
```

Now you can also access the CLI using the vty interface.
```bash
~$ telnet localhost 2601
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.

Hello, this is FRRouting (version 7.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.


User Access Verification

Password: zebra
Router> enable
Password: zebra
Router#
```

## Disable integrated configuration file (optional)
By default FRR uses an [Integrated configuration mode] where all configs are kept in a single `frr.conf` file.  Though 
optional, it is recommended to use a configuration file per daemon.  To disable the use of the integrated `frr.conf` and 
instead use a separate configuration file per daemon, first edit the `vtysh.conf`file.
```bash
sudo sed -i 's/^service/no service/g' /etc/frr/vtysh.conf
```

Now login with `vtysh` and save the configuration.
```bash
$ vtysh

Hello, this is FRRouting (version 7.1).
Copyright 1996-2005 Kunihiro Ishiguro, et al.

Router# wr mem
Note: this version of vtysh never writes vtysh.conf
Building Configuration...
Configuration saved to /etc/frr/zebra.conf
Configuration saved to /etc/frr/ripd.conf
Configuration saved to /etc/frr/ripngd.conf
Configuration saved to /etc/frr/ospfd.conf
Configuration saved to /etc/frr/ospf6d.conf
Configuration saved to /etc/frr/ldpd.conf
Configuration saved to /etc/frr/bgpd.conf
Configuration saved to /etc/frr/isisd.conf
Configuration saved to /etc/frr/pimd.conf
Configuration saved to /etc/frr/nhrpd.conf
Configuration saved to /etc/frr/eigrpd.conf
Configuration saved to /etc/frr/fabricd.conf
Configuration saved to /etc/frr/pbrd.conf
Configuration saved to /etc/frr/staticd.conf
Configuration saved to /etc/frr/bfdd.conf
```

Be sure to remove the combined configuration files.  All daemons check for the existence of this file at startup, and if 
it exists will not load their individual configuration files.
```bash
sudo rm -rf /etc/frr/frr.conf /etc/frr/frr.conf.sav
```

## Enable Eve-NG Telnet access to FRR nodes
You may prefer to connect to your new FRR instances using telnet instead of VNC.  This allows you to use client tools such
as Putty and SecureCRT.

**Warning !!! Use the following command INSIDE the FRR instance.  DO NOT execute this on your Eve-NG server or your workstation.**
```bash
sudo sed -i 's/GRUB_CMDLINE_LINUX=.*/GRUB_CMDLINE_LINUX="console=ttyS0,115200 console=tty0"/' /etc/default/grub
sudo update-grub
```

# Commit changes to qemu image
To preserve the installation and the default changes you've made you'll need to commit them to the base qemu image before
creating new instances of FRR inside of Eve-NG.
 
First shutdown the FRR node from inside the VNC connection.
```bash
sudo shutdown -h now
```

The changes we have made up to this point are all stored in a temporary directory inside Eve-NG.  To locate the directory,
on the left side-bar within the EVE lab interface, choose **Lab Details** to get your lab’s UUID.

![Lab Details](/assets/images/20191105/lab_details.png "Lab Details")

Make note of the **ID**. This will be part of the directory path to your temporary image file.

![Lab Details](/assets/images/20191105/lab_details_2.png "Lab Details")

> The UUID associated with my test lab is ID: 90d63733-6446-4437-a587-38050af45862

Next, find the POD ID of your user and the Node ID of your newly installed node.  From the left side menu select **Status**.

![Lab Status](/assets/images/20191105/lab_status.png "Lab Status")

The Status page will show your current POD ID.

![Lab Status](/assets/images/20191105/lab_status_1.png "Lab Status")

> Unless you have created multiple Eve-NG users the default Pod ID should be 0.

To get the Node ID right click on the node you created in the lab.

![Node Details](/assets/images/20191105/node_details.png "Node Details")

> If this is the first/only node you've created in this lab the ID should be 1.

Using this information, connect to your Eve-NG server and locate the directory that contains the changes you made to the 
default image.
```bash
cd /opt/unetlab/tmp/<pod_id>/<lab_uuid>/node_id/
```

Verify that you have found the correct image with;
```bash
~$ /opt/qemu/bin/qemu-img info virtioa.qcow2
image: virtioa.qcow2
file format: qcow2
virtual size: 16G (17179869184 bytes)
disk size: 3.3G
cluster_size: 65536
backing file: /opt/unetlab/addons/qemu/linux-frr-7.1/virtioa.qcow2
Format specific information:
    compat: 1.1
    lazy refcounts: false
    refcount bits: 16
    corrupt: false
```

> You should see that the backing file points to the original qemu image we created.

Now commit the changes back to the base image;
```bash
/opt/qemu/bin/qemu-img commit virtioa.qcow2
```

Once the command completes run the included Eve-NG script to check and fix file permissions.
```bash
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

# Testing new FRR images
Now that we have saved our FRR image we can launch multiple instances and create additional labs to test FRR's features.  I'll
demonstrate a simple OSPF routing lab and point out additional settings/options you may wish to use while running FRR in
Eve-NG.

## Add nodes
First we'll update the existing node in the lab we've already created.  Make sure the node has been stopped, right click
on the node and select **Edit**.

![Edit Node](/assets/images/20191105/edit_node.png "Edit Node")

Change the nodes name to something similar to `frr-01`.  Update the CPU and RAM settings as needed and change the number
of **Ethernets** to `3`.

![Edit Node](/assets/images/20191105/edit_node_1.png "Edit Node")

If you prefer to use Telnet instead of VNC, change the **Console** type to `telnet`.  Click **Save** to update the node.

![Edit Node](/assets/images/20191105/edit_node_2.png "Edit Node")

Now add an additional FRR node by right clicking in the open area of the lab and selecting **Node** as shown in
[previous steps](#create-eve-ng-lab).

![Add Node](/assets/images/20191105/add_2nd_node.png "Add Node")

Connect the second node to the existing cloud network using its `e0` interface.

![Connect Node](/assets/images/20191105/connect_2nd_node.png "Connect Node")

Connect the two FRR instances together by dragging the connection icon from one instance to the other.

![Connect Node](/assets/images/20191105/connect_2nd_node_1.png "Connect Node")

In the **Add Connection** window choose interface `e1` for both FRR instances.

![Connect Node](/assets/images/20191105/connect_2nd_node_2.png "Connect Node")

Now we'll add two [Virtual PC Simulator] (VPCS) instances to act as end hosts connected to our FRR OSPF network.  Right
click on the lab and select **Node** again.

![Add VPCS](/assets/images/20191105/add_vpcs.png "Add VPCS")

Scroll to the bottom of the list and select **Virtual PC (VPCS)**.

![Add VPCS](/assets/images/20191105/add_vpcs_1.png "Add VPCS")

In the **Add Node** window, set the number of nodes to add to `2`, set a name/prefix like `VPC` and click **Save**.

![Add VPCS](/assets/images/20191105/add_vpcs_2.png "Add VPCS")

Now connect each VPCS to a respective instance of FRR.

![Connect VPCS](/assets/images/20191105/connect_vpcs.png "Connect VPCS")

Connect the VPCS `eth0` interface to the FRR `e2` interface.

![Connect VPCS](/assets/images/20191105/connect_vpcs_1.png "Connect VPCS")

At this point your lab should look similar to the following.

![Final Lab](/assets/images/20191105/lab_final.png "Final Lab")

To start all nodes simultaneously, select **More Actions** from the left side menu.

![Start Nodes](/assets/images/20191105/start_all_nodes.png "Start Nodes")

Now select **Start all nodes** from the actions menu.

![Start Nodes](/assets/images/20191105/start_all_nodes_1.png "Start Nodes")

Now connect to all nodes by click on each inside the lab.

> All of my nodes are configured to use telnet and the Eve-NG client integration tools are configured to open my default 
>terminal application.

![Connect to Nodes](/assets/images/20191105/connect_to_all_nodes.png "Connect to Nodes")

## Configure OSPF Routing on FRR
Apply the following configurations to the FRR nodes in the lab.

**frr-01**

```bash
~$ vtysh

configure terminal
interface ens4
 no shutdown
 ip address 10.0.0.1/30
interface ens5
 no shutdown
 ip address 10.0.1.1/24
router ospf
 ospf router-id 1.1.1.1
 network 0.0.0.0/0 area 0.0.0.0
exit
exit
write memory
```

**frr-02**
```bash
~$ vtysh

configure terminal
interface ens4
 no shutdown
 ip address 10.0.0.2/30
interface ens5
 no shutdown
 ip address 10.0.2.1/24
router ospf
 ospf router-id 2.2.2.2
 network 0.0.0.0/0 area 0.0.0.0
exit
exit
write memory
```

## Configure VPCS Networking
Now add an IP and default gateway to each of the VPCS instances.

**vpcs1**
```bash
ip 10.0.1.10/24 10.0.1.1
````

**vpcs2**
```bash
ip 10.0.2.10/24 10.0.1.1
````

## Verify Routing
Now verify OSPF routing and reachability between the two VPCS instances.

**frr-01**
```bash
~$ sho ip ospf interface ens4
ens4 is up
  ifindex 3, MTU 1500 bytes, BW 4294967295 Mbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 10.0.0.1/30, Broadcast 10.0.0.3, Area 0.0.0.0
  MTU mismatch detection: enabled
  Router ID 1.1.1.1, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State Backup, Priority 1
  Backup Designated Router (ID) 1.1.1.1, Interface Address 10.0.0.1
  Multicast group memberships: OSPFAllRouters OSPFDesignatedRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 5.103s
  Neighbor Count is 1, Adjacent neighbor count is 1
```
```bash
~$ sho ip ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
2.2.2.2           1 Full/DR           38.920s 10.1.0.65       ens3:10.1.0.64           0     0     0
2.2.2.2           1 Full/DR           38.922s 10.0.0.2        ens4:10.0.0.1            0     0     0
```
```bash
~$ sho ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.0.0/30 [110/1] is directly connected, ens4, 00:02:08
O   10.0.1.0/24 [110/1] is directly connected, ens5, 00:05:23
O>* 10.0.2.0/24 [110/2] via 10.0.0.2, ens4, 00:01:58
  *                     via 10.1.0.65, ens3, 00:01:58
O   10.1.0.0/24 [110/1] is directly connected, ens3, 00:05:23
```

**frr-01**
```bash
~$ sho ip ospf interface ens4
ens4 is up
  ifindex 3, MTU 1500 bytes, BW 4294967295 Mbit <UP,BROADCAST,RUNNING,MULTICAST>
  Internet Address 10.0.0.2/30, Broadcast 10.0.0.3, Area 0.0.0.0
  MTU mismatch detection: enabled
  Router ID 2.2.2.2, Network Type BROADCAST, Cost: 1
  Transmit Delay is 1 sec, State DR, Priority 1
  Backup Designated Router (ID) 1.1.1.1, Interface Address 10.0.0.1
  Multicast group memberships: OSPFAllRouters OSPFDesignatedRouters
  Timer intervals configured, Hello 10s, Dead 40s, Wait 40s, Retransmit 5
    Hello due in 3.541s
  Neighbor Count is 1, Adjacent neighbor count is 1
```
```bash
~$ sho ip ospf neighbor 

Neighbor ID     Pri State           Dead Time Address         Interface            RXmtL RqstL DBsmL
1.1.1.1           1 Full/Backup       38.479s 10.1.0.64       ens3:10.1.0.65           0     0     0
1.1.1.1           1 Full/Backup       38.479s 10.0.0.1        ens4:10.0.0.2            0     0     0
```
```bash
~$ sho ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, D - SHARP,
       F - PBR, f - OpenFabric,
       > - selected route, * - FIB route, q - queued route, r - rejected route

O   10.0.0.0/30 [110/1] is directly connected, ens4, 00:22:33
O>* 10.0.1.0/24 [110/2] via 10.0.0.1, ens4, 00:22:23
  *                     via 10.1.0.62, ens3, 00:22:23
O   10.0.2.0/24 [110/1] is directly connected, ens5, 00:22:39
O   10.1.0.0/24 [110/1] is directly connected, ens3, 00:22:39
```

**vpcs1**
```bash
VPCS> ping 10.0.2.10   

84 bytes from 10.0.2.10 icmp_seq=1 ttl=62 time=3.041 ms
84 bytes from 10.0.2.10 icmp_seq=2 ttl=62 time=2.739 ms
84 bytes from 10.0.2.10 icmp_seq=3 ttl=62 time=4.734 ms
84 bytes from 10.0.2.10 icmp_seq=4 ttl=62 time=17.732 ms
84 bytes from 10.0.2.10 icmp_seq=5 ttl=62 time=2.129 ms
```

**vpcs2**
```bash
VPCS> ping 10.0.1.10

84 bytes from 10.0.1.10 icmp_seq=1 ttl=62 time=6.083 ms
84 bytes from 10.0.1.10 icmp_seq=2 ttl=62 time=2.504 ms
84 bytes from 10.0.1.10 icmp_seq=3 ttl=62 time=8.699 ms
84 bytes from 10.0.1.10 icmp_seq=4 ttl=62 time=2.528 ms
84 bytes from 10.0.1.10 icmp_seq=5 ttl=62 time=4.602 ms
```

# Conclusion
Now that you have tested the basic functionality of your new FRR image you can proceed to create more complex labs and test
additional features of FRR.  Whether you're new to open source routing or you have plans to write your own [Network
Operating System](https://en.wikipedia.org/wiki/Network_operating_system) (NOS) based on FRR, I recommend you check out 
the [participate](https://frrouting.org/#participate) section of the FRR website to find out how you can become more 
involved with the project.

[Zebra]:http://www.zebra.org/ 
[Quagga]:https://www.quagga.net/
[FRRouting]:https://frrouting.org/
[GPLv2+]:https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html
[The Linux Foundation]:https://www.linuxfoundation.org/
[KVM]:https://www.linux-kvm.org/page/Main_Page
[Eve-NG]:http://www.eve-ng.net/
[Releases]:https://github.com/FRRouting/frr/releases
[Supported Protocols]:http://docs.frrouting.org/en/latest/overview.html#supported-protocols-vs-platform
[Ubuntu Server]:https://ubuntu.com/download/server
[Linux How-To]:http://www.eve-ng.net/documentation/howto-s/106-howto-create-own-linux-image
[minimum requirements]:https://help.ubuntu.com/lts/serverguide/preparing-to-install.html#system-requirements
[qemu]:https://www.qemu.org/
[Grub]:https://www.gnu.org/software/grub/
[HWE Kernel Install]:https://itsfoss.com/ubuntu-hwe-kernel/
[LTS Enablement Stack]:https://wiki.ubuntu.com/Kernel/LTSEnablementStack
[vtysh]:http://docs.frrouting.org/projects/dev-guide/en/latest/vtysh.html#vtysh
[zebra]:http://docs.frrouting.org/en/latest/zebra.html
[staticd]:http://docs.frrouting.org/en/latest/static.html
[pthread]:https://en.wikipedia.org/wiki/POSIX_Threads
[MPLS]:http://docs.frrouting.org/en/latest/zebra.html#mpls-commands
[VRFs]:http://docs.frrouting.org/en/latest/zebra.html#virtual-routing-and-forwarding
[vty]:http://docs.frrouting.org/en/latest/basic.html#virtual-terminal-interfaces
[Virtual PC Simulator]:https://sourceforge.net/projects/vpcs/
[sysctl settings and kernel modules]:http://docs.frrouting.org/en/latest/installation.html#linux-sysctl-settings-and-kernel-modules
[Client Integration]:https://github.com/SmartFinn/eve-ng-integration
[VNC]:https://wiki.gnome.org/Apps/Vinagre
[Integrated configuration mode]:http://docs.frrouting.org/en/latest/vtysh.html#integrated-configuration-mode