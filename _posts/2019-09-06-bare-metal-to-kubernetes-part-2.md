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
![MAAS Inventory](/assets/images/20190906/maas_inventory.png "MAAS Inventory")

>To deploy Openstack you must have a bare minimum of two physical machines available in MAAS.  If you don't have an extra 
>machine to add to MAAS for the Juju controller you can deploy a [multi-cloud controller] on your workstation using [LXD].

[KVM Pod]:https://maas.io/docs/kvm-introduction
[prerequisites]:https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-1/#prerequisites
[multi-cloud controller]:https://jaas.ai/docs/clouds#heading--managing-multiple-clouds-with-one-controller
[LXD]:https://linuxcontainers.org/lxd/introduction/

# Install Juju
There are two methods of installing Juju on Debian based Linux, [Snapd] and [PPA].  Snap packages are also available for
other Linux distributions.  For MacOS and Windows reference the Juju installation [documentation].  I'll be using the snap 
package with a Debian based distribution of Linux.

[Snapd]:https://snapcraft.io/docs/installing-snapd
[PPA]:https://launchpad.net/~juju/+archive/ubuntu/stable
[documentation]:https://jaas.ai/docs/installing

To install the snap run the following
```bash
sudo snap install juju --classic
```

# Configure Juju
Before we can use Juju to model and deploy workloads we must first configure a few prerequisites.
* **Clouds** - Define the underlying provider of bare metal and/or virtual machines.
* **Credentials** - Keys, user names, passwords etc. used to authenticate with each cloud.
* **Controllers** - Machines deployed within a cloud to manage configuration and resources.
* **Models** - An environment associated with a controller in which workloads are deployed.

## Add Clouds
Juju works on top of many different types of [clouds] and has built in support for MAAS.  To see the available clouds run
the following from you workstation console;
```bash
~$ juju clouds --local

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
~$ juju add-cloud

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

## Add Credentials
As the output shows we'll need to add our MAAS server [credentials] before Juju can interact with the MAAS API.  To obtain
your MAAS API key, log in to MAAS and click on your username in the top right of the MAAS GUI.
![Obtain Credentials](/assets/images/20190906/obtain_credentials.png "Obtain Credentials")

[credentials]:https://jaas.ai/docs/credentials

Copy the API key shown or generate a new key.
![Copy Key](/assets/images/20190906/copy_key.png "Copy Key")

Now run the following and enter the copied API key when prompted.
```bash
~$ juju add-credential maas-01

Enter credential name: maas-01_creds

Using auth-type "oauth1".

Enter maas-oauth: 

Credential "maas-01_creds" added locally for cloud "maas-01".
```

To validate/list the added credentials run;
```bash
~$ juju credentials

Cloud    Credentials
maas-01  maas-01_creds
```

## Add Controllers
Now that we have added a cloud and its associated credentials we can add a Juju [controller].
>A controller is the management node of a Juju cloud environment. It runs the API server and the underlying database. 
>Its overall purpose is to keep state of all the models, applications, and machines in that environment.

[controller]:https://jaas.ai/docs/creating-a-controller

The controller only requires a minimum of 3.5 GiB of memory and 1vCPU so to ensure that Juju deploys the controller on the
virtual machine I've added to MAAS I'm going to add a [Tag] to the machine that Juju can use as a deployment [constraint].

[Tag]:https://jaas.ai/docs/constraints-reference
[constraint]:https://jaas.ai/docs/constraints

To add a Tag to a machine go to the Machines tab of the MAAS GUI and click on the name of the machine you would like to
tag.  Next click the **Edit** link under the **Tags** section.
![Add Tag](/assets/images/20190906/add_tag.png "Add Tag")

Click the **Edit** button to the right of the **Machine configuration**
![Edit Config](/assets/images/20190906/edit_config.png "Edit Config")

Add **juju** to the **Tags** section and click **Save changes**
![Save Tags](/assets/images/20190906/save_tags.png "Save Tags") 

To bootstrap the controller on MAAS execute the following from you workstation;
```bash
~$ juju bootstrap --constraints tags=juju maas-01 juju-01

Creating Juju controller "juju-01" on maas-01
Looking for packaged Juju agent version 2.6.8 for amd64
Launching controller instance(s) on maas-01...
 - eppk7s (arch=amd64 mem=4G cores=1)  
Installing Juju agent on bootstrap instance
Fetching Juju GUI 2.15.0
```

This will bootstrap a controller named `juju-01` to a machine with the tag `juju` on the `maas-01` cloud.  You can verify 
in MAAS that your machine is being deployed.
![Deploy Controller](/assets/images/20190906/deploy_controller.png "Deploy Controller")

After a few minutes your terminal will return the deployment status and you can verify the deployment with the `juju controllers`
command.
```bash
Waiting for address
Attempting to connect to 10.1.20.13:22
Connected to 10.1.20.13
Running machine configuration script...
Bootstrap agent now started
Contacting Juju controller at 10.1.20.13 to verify accessibility...

Bootstrap complete, controller "juju-01" now is available
Controller machines are in the "controller" model
Initial model "default" added

~$ juju controllers

Use --refresh option with this command to see the latest information.

Controller  Model    User   Access     Cloud/Region  Models  Nodes    HA  Version
juju-01*    default  admin  superuser  maas-01            2      1  none  2.6.8 
```

### Juju GUI
In addition to the API the Juju controller has a web interface that makes it easier to experiment with with Juju's modeling 
and automation capabilities.  To access the GUI run the following;
```bash
~$ juju gui

GUI 2.15.0 for model "admin/default" is enabled at:
  https://10.1.20.13:17070/gui/u/admin/default
Your login credential is:
  username: admin
  password: 85faa8d324ce26300d8aa6b43b8fd794
```

Open a browser and enter the url and credentials provided in the output.

![Juju GUI](/assets/images/20190906/juju_gui.png "Juju GUI")

You can play around with the default model from here, click the green circle in the center of the model to add a workload 
or use the search in the top right to explore the available [Charms] in the Charm [Store].
![Juju GUI](/assets/images/20190906/juju_gui_2.png "Juju GUI")

[Charms]:https://jaas.ai/docs/charm-writing
[Store]:https://jaas.ai/store

>The rest of this demonstration will use the controller API however it is useful to monitor the status of your models in 
>the GUI as you continue the deployment.

## Add Models
When a controller is bootstrapped two [models] are automatically added, **controller** and **default**.  The controller 
model is used for internal Juju management and is not intended for general workloads. The default model can be used to
deploy any supported software but is typically used for experimentation purposes.

[models]:https://jaas.ai/docs/configuring-models

Before deploying Openstack we'll create a new model by running the following;
```bash
~$ juju add-model openstack

Added 'openstack' model with credential 'maas-01_creds' for user 'admin'
```

To verify that the model was created and selected as the active model run;
```bash
~$ juju models

Controller: juju-01

Model       Cloud/Region  Type  Status     Machines  Cores  Access  Last connection
controller  maas-01       maas  available         1      1  admin   just now
default     maas-01       maas  available         0      -  admin   5 minutes ago
openstack*  maas-01       maas  available         0      -  admin   never connected
```

# Deploy Openstack
Both the [Openstack] and [Juju] documentation contain detailed, step by step instructions for installing the core components 
of Openstack with Juju.  Instead of just providing those as reference I wanted to show how you can customize the 
installation to reduce the number of servers needed as well as how to include [Octavia] (Openstacks' LBaaS implementation).  In
my next post I'll demonstrate how Kubernetes interacts directly with Octavia to provide L4 load balancing services for 
container workloads.

>The Juju and Openstack documentation both utilize a minimum of four servers in their examples.  My demonstration will show
>how to deploy on a minimum of two servers.  Deploying on a single server is only possible with [LXD] as [Neutron Gateway] 
>and [Nova Compute] can not run on the same bare metal server.

[Openstack]:https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/install-openstack.html#deploy-openstack
[Juju]:https://jaas.ai/openstack-base/bundle/61
[Octavia]:https://jaas.ai/octavia/11
[Neutron Gateway]:https://jaas.ai/neutron-gateway
[Nova Compute]:https://jaas.ai/nova-compute

## Configure Openstack Bundle
Before we deploy the [Openstack Base] bundle we need to make modifications to match our lab environment.  There's a charm 
development tool aptly named _charm_ that can be installed as a snap.  To install the snap run;
```bash
sudo snap install charm --classic
```
For other distributions please reference the Charm Tools Project [documentation](https://jaas.ai/docs/charm-tools).

[Openstack Base]:https://jaas.ai/openstack-base/bundle/61

Now we'll pull down the current version of the Openstack bundle with;
```bash
charm pull cs:openstack-base
```

Change to the newly created directory and create a backup of the bundle.yaml file;
```bash
cd openstack-base
cp bundle.yaml bundle.yaml.bak
```

If you've been following my examples you can download a copy of the bundle file I'm using or edit the bundle.yaml file to
match your own environment;
```bash
wget https://gist.githubusercontent.com/2stacks/d0b4b4b81df4a835934bbd9b8543ad2e/raw/8a8bc1c3293222a179683ba62092224a1b4567ac/bundle.yaml
```

You can diff my version with the original file to see what I've changed.  
```bash
diff bundle.yaml bundle.yaml.bak
```
In short, I'm placing Neutron Gateway and all of the other Openstack dependencies on one host and reserving the remaining 
host for Nova Compute.  Remember that Neutron Gateway and Nova Compute can not run on the same bare metal machine.  In 
order to have as much capacity as possible for running virtual machines I'm not running any additional services on the 
host dedicated to Nova Compute.

## Configure Octavia Overlay
Next we'll download and modify the Octavia overlay used to deploy the resources required by Octavia.
```bash
curl https://raw.githubusercontent.com/openstack-charmers/openstack-bundles/master/stable/overlays/loadbalancer-octavia.yaml -o loadbalancer-octavia.yaml
```

By default this overlay expects a minimum of four machines so it will need to be modified to run with fewer machines.  As
before I've modified the file to deploy all of the resources on the host running Neutron Gateway.  You can download my 
version of the file by running;
```bash
mv loadbalancer-octavia.yaml loadbalancer-octavia.yaml.bak
wget https://gist.githubusercontent.com/2stacks/21ba79de48e38b3588868cd033675d1a/raw/2eac80acffc6173b996cc30862da7a02317ce9a3/loadbalancer-octavia.yaml
```

## Deploy Openstack Bundle
Once we've made the necessary changes or downloaded the modified files we can start the deployment of Openstack.
```bash
juju deploy ./bundle.yaml --overlay loadbalancer-octavia.yaml
```

The deployment time depends on the speed of your hardware but in my environment it takes approximately 15 minutes.  You 
can monitor the deployment status by running the following command.  
```bash
watch -c juju status --color
```
You can also monitor the status of the deployment and view the model of the Openstack installation that has been constructed 
in the [Juju GUI](#juju-gui).  While Juju is deploying the Openstack workloads and resources we can continue to apply 
additional configurations required by Octavia.

## Configure Octavia
Octavia consists of an Openstack controller and load balancer instances that deploy as virtual machines on Nova Compute.  The 
communication between the controller and each instance is secured with bi-directional certificate authentication over an 
out-of-band management network.  The certificate, network and image resources must be configured before you can deploy 
Octavia load balancer instances.

>For additional information on configuring the Octavia Charm please reference the [documentation](https://jaas.ai/octavia/11) 

### Generate Certificates
Below is an example of how to create self-signed certificates for use by Octavia.  **Note:** This example is not meant to 
be used in production.
```bash
mkdir -p demoCA/newcerts
touch demoCA/index.txt
touch demoCA/index.txt.attr
openssl genrsa -passout pass:foobar -des3 -out issuing_ca_key.pem 2048
openssl req -x509 -passin pass:foobar -new -nodes -key issuing_ca_key.pem \
    -config /etc/ssl/openssl.cnf \
    -subj "/C=US/ST=Somestate/O=Org/CN=www.example.com" \
    -days 365 \
    -out issuing_ca.pem
openssl genrsa -passout pass:foobar -des3 -out controller_ca_key.pem 2048
openssl req -x509 -passin pass:foobar -new -nodes \
    -key controller_ca_key.pem \
    -config /etc/ssl/openssl.cnf \
    -subj "/C=US/ST=Somestate/O=Org/CN=www.example.com" \
    -days 365 \
    -out controller_ca.pem
openssl req \
    -newkey rsa:2048 -nodes -keyout controller_key.pem \
    -subj "/C=US/ST=Somestate/O=Org/CN=www.example.com" \
    -out controller.csr
openssl ca -passin pass:foobar -config /etc/ssl/openssl.cnf \
    -cert controller_ca.pem -keyfile controller_ca_key.pem \
    -create_serial -batch \
    -in controller.csr -days 365 -out controller_cert.pem
cat controller_cert.pem controller_key.pem > controller_cert_bundle.pem
```

### Apply Certificates
To apply the certificates to the Octavia controller run the following command.
```bash
juju config octavia \
    lb-mgmt-issuing-cacert="$(base64 controller_ca.pem)" \
    lb-mgmt-issuing-ca-private-key="$(base64 controller_ca_key.pem)" \
    lb-mgmt-issuing-ca-key-passphrase=foobar \
    lb-mgmt-controller-cacert="$(base64 controller_ca.pem)" \
    lb-mgmt-controller-cert="$(base64 controller_cert_bundle.pem)"
```

### Create Openstack Resources
To initiate the creation of the Openstack network, router and security group used by Octavia instances run;
```bash
juju run-action --wait octavia/0 configure-resources
```

### Deploy Amphora Image
Finally to create the disk image that will be used by load balancer instances run the following.
```bash
juju run-action --wait octavia-diskimage-retrofit/leader retrofit-image
```
>By default this will create an Ubuntu bionic image preconfigured with [HAProxy](http://www.haproxy.org/).This command 
>can take a long time to finish so be patient.  It should eventually create an image with a prefix of _amphora-haproxy_. We'll 
>validate that this image was successfully created in the next section.

# Validate Deployment
Once everything has been configured and Juju has finished it's deployment we can begin validating the installation by creating
resources within Openstack.

>The output of `juju status` should show everything as **active** with the exception of Vault.  It's ok to leave this as 
>is for now but if you'd like additional information on setting up Vault please reference [Appendix C] of the Openstack 
>deployment documentation.

[Appendix C]:https://docs.openstack.org/project-deploy-guide/charm-deployment-guide/latest/app-vault.html

## API Access
The Openstack installation should now be accessible via it's API and built in web interface.  To begin utilizing the API
we need to install the [OpenStackClient] package.

>OpenStackClient (aka OSC) is a command-line client for OpenStack that brings the command set for Compute, Identity, Image, 
>Object Storage and Block Storage APIs together in a single shell with a uniform command structure.

[OpenStackClient]:https://docs.openstack.org/python-openstackclient/pike/

There are several ways to install the client package but for most linux distributions the snap package is recommended.
```bash
sudo snap install openstackclients
```

The credentials required to access the API can be obtained and set as environment variables by running the rc script included
in the **openstack-base** directory.
```bash
source openrc
```

To view the credentials run;
```bash
~$ env | grep OS_

OS_REGION_NAME=RegionOne
OS_USER_DOMAIN_NAME=admin_domain
OS_PROJECT_NAME=admin
OS_AUTH_VERSION=3
OS_IDENTITY_API_VERSION=3
OS_PASSWORD=MaemumeFoolah1ae
OS_AUTH_TYPE=password
OS_AUTH_URL=http://10.1.20.36:5000/v3
OS_USERNAME=admin
OS_PROJECT_DOMAIN_NAME=admin_domain
```

To verify access to the API run the following commands.  They should output a list of all of the Openstack API endpoints 
and services created by Juju.
```bash
openstack catalog list
openstack endpoint list
```

## Web Dashboard Access
To access the Openstack web interface, retrieve the IP address of the LXD container that the Openstack Dashboard was deployed on.
```bash
juju status openstack-dashboard/0* | grep Container | awk '{print $3}'
```

Open a browser and use the returned IP address in the following URL;

`http://<dashboard_ip>/horizon`

The credentials needed to log in to the browser are the same as previously obtained with the **openrc** script.
```bash
OS_USER_DOMAIN_NAME=admin_domain
OS_PROJECT_NAME=admin
OS_PASSWORD=MaemumeFoolah1ae
```

![Openstack GUI](/assets/images/20190906/openstack_gui.png "Openstack GUI")

## Add Images
During deployment the Juju [glance-simplestreams] charm should have added a few default disk images in addition to the 
[Amphora] image used by Octavia.  To verify that these images were created run the following;
```bash
~$ openstack image list

+--------------------------------------+-----------------------------------------------------------------+--------+
| ID                                   | Name                                                            | Status |
+--------------------------------------+-----------------------------------------------------------------+--------+
| 86c318f3-76f7-4311-b146-cc00e79e9406 | amphora-haproxy-x86_64-ubuntu-18.04-20190813.1                  | active |
| 40aca86a-9177-4b94-afe3-fee448abee4c | auto-sync/ubuntu-bionic-18.04-amd64-server-20190813.1-disk1.img | active |
| 4b238000-ba59-4470-bcea-93dd8761c40e | auto-sync/ubuntu-trusty-14.04-amd64-server-20190514-disk1.img   | active |
| d87ed10a-5d86-409c-9fa5-7cf59e80a258 | auto-sync/ubuntu-xenial-16.04-amd64-server-20190814-disk1.img   | active |
+--------------------------------------+-----------------------------------------------------------------+--------+
```
[glance-simplestreams]:https://jaas.ai/glance-simplestreams-sync/21
[Amphora]:https://docs.openstack.org/octavia/stein/reference/glossary.html

You can use the **auto-sync** images in the following examples or create your own via the API.  The following command will
create an image named _bionic_ using the current Ubuntu 18.04 cloud-image.
```bash
curl http://cloud-images.ubuntu.com/bionic/current/bionic-server-cloudimg-amd64.img | \
openstack image create --public --min-disk 3 --container-format bare --disk-format qcow2 \
    --property architecture=x86_64 --property hw_disk_bus=virtio --property hw_vif_model=virtio bionic
```

If you run `openstack image list` again you should see the newly created image in the list.
```bash
~$ openstack image list

+--------------------------------------+-----------------------------------------------------------------+--------+
| ID                                   | Name                                                            | Status |
+--------------------------------------+-----------------------------------------------------------------+--------+
| 86c318f3-76f7-4311-b146-cc00e79e9406 | amphora-haproxy-x86_64-ubuntu-18.04-20190813.1                  | active |
| 40aca86a-9177-4b94-afe3-fee448abee4c | auto-sync/ubuntu-bionic-18.04-amd64-server-20190813.1-disk1.img | active |
| 4b238000-ba59-4470-bcea-93dd8761c40e | auto-sync/ubuntu-trusty-14.04-amd64-server-20190514-disk1.img   | active |
| d87ed10a-5d86-409c-9fa5-7cf59e80a258 | auto-sync/ubuntu-xenial-16.04-amd64-server-20190814-disk1.img   | active |
| b2797730-70ca-4e91-8b30-26cf2e6c7968 | bionic                                                          | active |
+--------------------------------------+-----------------------------------------------------------------+--------+
```

## Add Flavors
Next we'll create [flavors] that determine the amount of resources given to virtual machine instances.  
```bash
openstack flavor create --ram 1024 --disk 10 m1.small
openstack flavor create --vcpus 2 --ram 2048 --disk 20 m1.medium
openstack flavor create --vcpus 4 --ram 4096 --disk 20 m1.large
```

[flavors]:https://docs.openstack.org/nova/stein/user/flavors.html

To list the newly created instance flavors run;
```bash
~$ openstack flavor list 

+--------------------------------------+-----------+------+------+-----------+-------+-----------+
| ID                                   | Name      |  RAM | Disk | Ephemeral | VCPUs | Is Public |
+--------------------------------------+-----------+------+------+-----------+-------+-----------+
| 8dc59145-1ba3-4709-b037-6806a38d598c | m1.large  | 4096 |   20 |         0 |     4 | True      |
| c14adc69-d72a-4ed2-b49a-d010a5e58fa3 | m1.small  | 1024 |   10 |         0 |     1 | True      |
| e426aa37-2d29-4ece-8e3c-c1fb833f9308 | m1.medium | 2048 |   20 |         0 |     2 | True      |
+--------------------------------------+-----------+------+------+-----------+-------+-----------+
```

## Add Networks and Subnets
We need to create at least two networks (internal and external) to verify the functionality of Openstack virtual routers 
and load balancers.  The two types of Openstack networks we'll create are;

* [provider-network]
* [self-service network]  

The first network type we'll create is a provider-network.  This network will provide ingress and egress 
communication between compute instances and the outside world.  Subnets created within this network can be used to assign 
floating IPs which will allow you to run public services and remotely reach your Openstack instances.

[provider-network]:https://docs.openstack.org/mitaka/networking-guide/intro-os-networking.html#provider-networks

To create the network run the following;
```bash
openstack network create ext_net --share --external --provider-network-type flat --provider-physical-network physnet1
```

To create a subnet inside the network for allocation of floating IPs run;
```bash
openstack subnet create ext_subnet --no-dhcp --allocation-pool start=192.168.10.101,end=192.168.10.254 \
    --subnet-range 192.168.10.0/24 --gateway 192.168.10.1 --dns-nameserver 192.168.10.1 --network ext_net
```

>Refer to [Part 1]({% post_url 2019-08-05-bare-metal-to-kubernetes-part-1 %}) of this series for the networks and interfaces
>we created to support the Openstack installation.  Note: additional subnets can be created within a network (ex. IPv6) if
>the underlying physical network supports it.

The second network type we'll create is a self-service network.  These networks allow non privileged users of the Openstack
installation to create project specific networks.  Typically they employ GRE or VXLAN as overlays and require an Openstack 
virtual router to connect to external networks.

[self-service network]:https://docs.openstack.org/mitaka/networking-guide/intro-os-networking.html#self-service-networks

To create the network run the following;
```bash
openstack network create int_net
```

To create a subnet inside the network;
```bash
openstack subnet create int_subnet --allocation-pool start=10.0.0.11,end=10.0.0.254 \
    --subnet-range 10.0.0.0/24 --gateway 10.0.0.1 --dns-nameserver 192.168.10.1 --network int_net
```

> This will deploy a virtual network within the default _admin_ project using the default overlay type of GRE.  Since this 
>subnet is not exposed outside of Openstack by default, you can configure any subnet and IP range you like.  Note that I 
>have configured a dns-nameserver external to Openstack for external name resolution.

To verify the networks and subnets we just created run;
```bash
~$ openstack network list

+--------------------------------------+-------------+--------------------------------------+
| ID                                   | Name        | Subnets                              |
+--------------------------------------+-------------+--------------------------------------+
| 1730fd08-c48c-4591-993c-7d5f90ea5578 | ext_net     | 3d283801-faee-44c1-b51b-5c69f76ef61f |
| 695d5dab-a12a-4471-841b-566c08ea732d | lb-mgmt-net | 525306b4-5653-4b80-bbe9-facf3d900bca |
| a606266e-3c10-4962-9588-71a7989c78ac | int_net     | c4a1bfd3-463d-44a0-b4ca-9748e76f90e6 |
+--------------------------------------+-------------+--------------------------------------+

~$ openstack subnet list

+--------------------------------------+------------------+--------------------------------------+-------------------------+
| ID                                   | Name             | Network                              | Subnet                  |
+--------------------------------------+------------------+--------------------------------------+-------------------------+
| 3d283801-faee-44c1-b51b-5c69f76ef61f | ext_subnet       | 1730fd08-c48c-4591-993c-7d5f90ea5578 | 192.168.10.0/24         |
| 525306b4-5653-4b80-bbe9-facf3d900bca | lb-mgmt-subnetv6 | 695d5dab-a12a-4471-841b-566c08ea732d | fc00:566c:8ea:732d::/64 |
| c4a1bfd3-463d-44a0-b4ca-9748e76f90e6 | int_subnet       | a606266e-3c10-4962-9588-71a7989c78ac | 10.0.0.0/24             |
+--------------------------------------+------------------+--------------------------------------+-------------------------+
```

For additional network options and settings consult the [Openstack Networking] documentation or run;
```bash
openstack network create --help
```

[Openstack Networking]:https://docs.openstack.org/neutron/stein/admin/

## Add a Router
Openstack virtual routers provide layer-3 services such as routing and NAT.  They are required for routing between 
self-service networks within the same project as well as for communication between self-service and provider networks.

To create a router within the current domain/project run;
```bash
openstack router create rtr-01
```

To attach a router interface to the previously created subnet within our provider network run;
```bash
openstack router set rtr-01 --external-gateway ext_net
```

And finally to connect our internal self-service subnet to the external provider subnet run;
```bash
openstack router add subnet rtr-01 int_subnet
```

To verify that the router was created and configured run;
```bash
openstack router list
openstack router show rtr-01
```

You can also verify via the Openstack web interface by navigating to; 

`http://<dashboard_ip>/horizon/project/network_topology/`

Here openstack will display a topology of the existing routers/networks/instances created within the current project.
![Network Topology](/assets/images/20190906/network_topology.png "Network Topology")

If all has gone well up to now you should be able to ping the external interface of the router from your workstation.  To
get the external IP of the router run;
```bash
router_ip=$(openstack port list --router rtr-01 --network ext_net -f value | awk -F "'" '{print $2}')
```

Now try pinging the returned IP from your workstation.
```bash
~$ ping $router_ip

PING 192.168.10.214 (192.168.10.214) 56(84) bytes of data.
64 bytes from 192.168.10.214: icmp_seq=1 ttl=64 time=1.17 ms
64 bytes from 192.168.10.214: icmp_seq=2 ttl=64 time=0.292 ms
```

## Add Security Groups
To allow access to our compute instances we'll need to create security groups or add rules to the existing security groups 
that were created during the installation of Openstack.  To view a list of the existing security groups run;
```bash
openstack security group list
```

Next we'll add rules to the default security group to allow ICMP and SSH access from external hosts.
```bash
project_id=$(openstack project show --domain admin_domain admin -f value -c id)
secgroup_id=$(openstack security group list --project ${project_id} | awk '/default/ {print $2}')
openstack security group rule create ${secgroup_id} --protocol icmp --remote-ip 0.0.0.0/0
openstack security group rule create ${secgroup_id} --protocol tcp --remote-ip 0.0.0.0/0 --dst-port 22
```

You can validate that the rules were added to the security group by running;
```bash
openstack security group show ${secgroup_id}
```

## Add SSH Keys
To be able to access our compute instances remotely via ssh we'll need to either create a new SSH keypair or upload the 
public key of an existing keypair.

To create a new keypair run;
```bash
touch ~/.ssh/os_rsa
chmod 600 ~/.ssh/os_rsa
openstack keypair create os_rsa > ~/.ssh/os_rsa
```

To upload an existing public key run;
```bash
openstack keypair create --public-key ~/.ssh/id_rsa.pub id_rsa
```

## Add an Instance
The following command will add a new compute instance named _bionic-test_ using the Ubuntu 18.04 cloud image.  The configured
flavor will give it 1 vCPU and 1G of RAM.  It will be configured with an IP in the internal subnet we created, be 
assigned to the default security group and will allow SSH access using the id_rsa keypair. 
```bash
openstack server create \
    --image bionic --flavor m1.small --key-name id_rsa \
    --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') \
    bionic-test
```

### Add Floating IP
To enable SSH access from external hosts we'll need to assign the instance a floating ip in the external subnet. 
```bash
floating_ip=$(openstack floating ip create -f value -c floating_ip_address ext_net)
openstack server add floating ip bionic-test ${floating_ip}
```

### Test Reachability
Once the **Status** of `openstack server list` shows that the instance is **ACTIVE** we can attempt to access it via ssh.
```bash
~$ openstack server list

+--------------------------------------+-------------+--------+-------------------+--------+----------+
| ID                                   | Name        | Status | Networks          | Image  | Flavor   |
+--------------------------------------+-------------+--------+-------------------+--------+----------+
| 946b52a2-bf60-4063-966c-f0d1f4828777 | bionic-test | ACTIVE | int_net=10.0.0.35 | bionic | m1.small |
+--------------------------------------+-------------+--------+-------------------+--------+----------+
```

```bash
~$ ssh -i ~/.ssh/id_rsa ubuntu@$floating_ip

Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-60-generic x86_64)

ubuntu@bionic-test:~$
```

## Add a Load Balancer
The last thing we'll test is the functionality of the Octavia load balancing service.  I'm not going set up multiple web servers
and verify full functionality for now.  The goal of this test is to verify than an Amphora instance is created and
that it can properly route traffic to the Ubuntu instance we just crated.

To create the load balancer with an interface in our internal subnet run the following;
```bash
lb_vip_port_id=$(openstack loadbalancer create -f value -c vip_port_id --name lb-01 --vip-subnet-id int_subnet)
```

Continue to run the following until lb-01 shows status of **ACTIVE** and **ONLINE**.
```bash
~$ openstack loadbalancer show lb-01
```

The instance can take some time to create so be patient.  If the image creation fails you can check the worker logs on the
Octavia controller.
```bash
juju ssh octavia/0
more /var/log/octavia/octavia-worker.log 
```

### Configure the Load Balancer
Once the load balancer has been created we can configure vips/pools/monitors etc.  I'm only going to show how to create a
simple service to forward ssh connections to the _bionic-test_ instance we created.
```bash
openstack loadbalancer listener create --name listener1 --protocol tcp --protocol-port 22 lb-01
openstack loadbalancer pool create --name pool1 --lb-algorithm ROUND_ROBIN --listener listener1 --protocol tcp
openstack loadbalancer healthmonitor create --delay 5 --timeout 5 --max-retries 3 --type TCP pool1
instance_ip=$(openstack server show bionic-test -f value -c addresses | sed -e 's/int_net=//' | awk -F "," '{print $1}')
openstack loadbalancer member create --subnet-id int_subnet --address ${instance_ip} --protocol-port 22 pool1
```

Now verify the status of the health monitor and pool member.
```bash
~$ openstack loadbalancer member list pool1

+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| id                                   | name | project_id                       | provisioning_status | address   | protocol_port | operating_status | weight |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
| b332c304-3695-4e83-a57f-7bcd2bf76f3d |      | 55d6e3168bc24e0397c1185a09500b78 | ACTIVE              | 10.0.0.35 |            22 | ONLINE           |      1 |
+--------------------------------------+------+----------------------------------+---------------------+-----------+---------------+------------------+--------+
```

### Add Floating IP
To reach our load balancer externaly we'll need to create and assign a floating IP address.
```bash
floating_ip=$(openstack floating ip create -f value -c floating_ip_address ext_net)
openstack floating ip set --port $lb_vip_port_id $floating_ip
```

### Test Reachability
Now ssh to the newly created floating IP.  If the load balancer was created correctly it should forward your connection 
to your internal instance.

```bash
~$ ssh -i ~/.ssh/id_rsa ubuntu@$floating_ip

Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-60-generic x86_64)

ubuntu@bionic-test:~$
``` 

# Conclusion
At this point we have deployed and validated the essential components of Openstack necessary to complete our 
Kubernetes-as-a-Service solution.  Openstack is a very robust and capable IaaS solution for building public and private 
clouds.  This post is only intended to demonstrate a subset of it's capabilities and how to use open source orchestration
to simplify what is ordinarily a very complex and labor intensive endeavor.  In my next post I'll continue with this trend of 
simplifying complex deployments through the use of open source solutions and demonstrate how to automate Kubernetes
deployments on top of Openstack.