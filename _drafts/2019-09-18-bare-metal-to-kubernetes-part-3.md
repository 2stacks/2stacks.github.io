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
I started this series of posts with the assertion that, although a lot of effort has gone in to making Kubernetes easier
to deploy and operate, Kubernetes is still hard.  Simply showing how to deploy a Kubernetes cluster is not the 
aim.  There are thousands of how-to's on the web covering the basics.  The ultimate point of these posts is to demonstrate 
at least one viable option for deploying a capability similar to a public cloud Kubernetes-as-a-service solution.  This 
means having the ability to provision multiple, production-ready clusters complete with features such as [multitenancy], 
role based access control ([RBAC]), [service mesh], [CI/CD] etc.  When you hear "Kubernetes is hard" it's not usually in 
reference to simply setting up a functioning cluster.  It is usually a commentary on the challenges associated with these 
additional features which are necessary for running Kubernetes in production.

# Objective
I mentioned in [Part 1]({% post_url 2019-08-05-bare-metal-to-kubernetes-part-1 %}) of this series some of the available 
solutions for deploying Kubernetes in private clouds.  After some extensive research and experimentation with a few of 
the more popular solutions I decided to go with [Rancher's] Kubernetes management platform.  This is not an endorsement of 
one product over another and I don't intend to do a side-by-side comparison of all of the platforms I tested.  For me,
Rancher simply shortened the steep learning curve associated with Kubernetes and met or exceeded all of the requirements 
I had for running production clusters with limited experience.  

In this final post I'm going to demonstrate how to integrate Rancher with Openstack.  Integrating these two technologies 
simplifies the deployment and management of production ready Kubernetes clusters.  Rancher abstracts and automates the 
process of creating the underlying infrastructure and provides a unified management platform for running Kubernetes at 
scale on any cloud, public or private.  Once completing this demonstration we'll have a self-service platform from which 
to create and manage multiple Kubernetes clusters with the features necessary for production environments.  

[RBAC]:https://en.wikipedia.org/wiki/Role-based_access_control
[service mesh]:https://istio.io/docs/concepts/what-is-istio/#what-is-a-service-mesh
[CI/CD]:https://en.wikipedia.org/wiki/CI/CD
[multitenancy]:https://en.wikipedia.org/wiki/Multitenancy
[Rancher's]:https://rancher.com/

# Prerequisites
In [Part 1]({% post_url 2019-08-05-bare-metal-to-kubernetes-part-1 %}) of this series we deployed [MAAS] to automate the
provisioning and management of our physical compute resources.  In [Part 2]({% post_url 2019-09-06-bare-metal-to-kubernetes-part-2 %})
we deployed [Openstack] as our Infrastructure-as-a-Service (IaaS) platform for managing virtual machines, networks and
storage.  These prerequisites provide the private cloud infrastructure on which Rancher will automate self-service Kubernetes
deployments.  If you've been following along up till now you should already have everything you need to complete this final
demonstration. 

[MAAS]:https://maas.io/
[Openstack]:https://www.openstack.org/

# Install Rancher
Rancher was designed to be [cloud-native] and is intended to be run as a distributed, highly available (HA) application on top of
Kubernetes.  That said, getting started with Rancher is as simple as launching a single [container].  For the purposes of this demonstration I'll be using a 
single node deployment.  For additional information on running Rancher in HA please reference the [Rancher Documentation].

[cloud-native]:https://opensource.com/article/18/7/what-are-cloud-native-apps
[container]:https://techcrunch.com/2016/10/16/wtf-is-a-container/
[Rancher Documentation]:https://rancher.com/docs/rancher/v2.x/en/

## Create Cloud-init file
To automate the creation and installation of the Rancher server we'll use a [cloud-init] file with the Openstack API.

>Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialization. It is 
>supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and bare-metal 
>installations.

[cloud-init]:https://cloudinit.readthedocs.io/en/latest/

First we'll need to retrieve the SSH public key from the keypair that was created in [Part 2](https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-2/#add-ssh-keys)
```bash
openstack keypair show --public-key os_rsa
```

Next using your preferred text editor create a file named `cloud-init` with the following contents.  Be sure to replace
`<your public key>` with the public key retrieved in the previous step.
```bash
#cloud-config
groups:
  - ubuntu
  - docker
users:
  - name: ubuntu
    shell: /bin/bash
    sudo: ['ALL=(ALL) NOPASSWD:ALL']
    primary-group: ubuntu
    groups: sudo, docker
    lock_passwd: true
    ssh-authorized-keys:
      - ssh-rsa <your public key>
apt:
  sources:
    docker:
      keyid: "0EBFCD88"
      source: "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable" 
package_upgrade: true
packages: 
  - apt-transport-https 
  - docker-ce
runcmd:
  - "docker run -d --restart=unless-stopped -p 80:80 -p 443:443 rancher/rancher"
```
This cloud-init file will instruct the Ubuntu virtual machine, that we'll launch in the next step, to install Docker, then
pull and run the Rancher server container.

## Create Rancher Server
You should have already created the required Openstack flavors, images and networks needed to launch the server.  To launch
the server with the custom cloud-init file run; 
```bash
openstack server create --image bionic --flavor m1.medium \
    --nic net-id=$(openstack network list | grep int_net | awk '{ print $2 }') \
    --user-data cloud-init rancher
```

## Add Floating IP
To enable SSH access from external hosts we'll need to assign the instance a floating ip in the external subnet. 
```bash
floating_ip=$(openstack floating ip create -f value -c floating_ip_address ext_net)
openstack server add floating ip rancher ${floating_ip}
```

# Security Groups
Rancher and Kubernetes in general has many port and protocol requirements for communicating with cluster nodes.  There are 
several different ways to implement these requirements depending upon the type of public or private cloud you are deploying 
to.  Make sure you review these [port and protocol] requirements before running Rancher in production.  For demonstration 
purposes I'm going to update the existing default security group in Openstack to allow all traffic.  It should go without 
saying that this is not recommended outside of a lab environment.

[port and protocol]:https://rancher.com/docs/rancher/v2.x/en/installation/requirements/

To update the default security group run;
```bash
project_id=$(openstack project show --domain admin_domain admin -f value -c id)
secgroup_id=$(openstack security group list --project ${project_id} | awk '/default/ {print $2}')
openstack security group rule create $secgroup_id --protocol any --ethertype IPv6 --ingress
openstack security group rule create $secgroup_id --protocol any --ethertype IPv4 --ingress
```

# Connect to Rancher UI
Our Rancher server should now be remotely reachable via the floating IP address we added earlier.  To get the IP run;
```bash
openstack server show rancher -f value -c addresses
```

Now open a web browser and launch the Rancher User Interface (UI) at `https://<floating_ip>`.

![Rancher UI](/assets/images/20190918/rancher_ui.png "Rancher UI")

# Configure Rancher
Upon connecting to the Rancher UI you'll be prompted to set a new password for the default `admin` user.

After selecting **Continue** you'll be prompted to set the Rancher server URL.  It's OK to leave the default for now but 
typically this would be the fully qualified domain name of the server.  Note: all cluster nodes will
need to be able to reach this URL and it can be changed later from within the UI.

![Rancher URL](/assets/images/20190918/rancher_url.png "Rancher URL")

After selecting **Save URL** you'll be logged in to the default **Clusters** view of the UI.  Since no clusters have been
added, only a **Add Cluster** button is visible.  

## Enable Openstack Node Driver
Before adding our first cluster we need to configure settings to allow Rancher to automate the provisioning of Openstack 
resources.  The first configuration change will be to enable the Openstack [Node Driver].

>Node drivers are used to provision hosts, which Rancher uses to launch and manage Kubernetes clusters.

[Node Driver]:https://rancher.com/docs/rancher/v2.x/en/admin-settings/drivers/node-drivers/

To enable the Openstack Node Driver navigate to **Tools** then **Drivers** in the UI navigation.

![Enable Node Driver](/assets/images/20190918/node_driver_1.png "Enable Node Driver")

On the Drivers page, click tab for **Node Drivers**, select the check box next to **OpenStack** and then click the **Activate**
button.

![Enable Node Driver](/assets/images/20190918/node_driver_2.png "Enable Node Driver")

## Add Openstack Node Template
Once we've enabled the Openstack Node Driver we can create a [Node Template] that will specify the settings used to create
nodes/hosts within Openstack.

[Node Template]:https://rancher.com/docs/rancher/v2.x/en/cluster-provisioning/rke-clusters/node-pools/#node-templates

To create the node template click the drop down in the top right of the UI next to the admin user avatar and select 
**Node Templates**.

![Enable Node Template](/assets/images/20190918/node_template_1.png "Enable Node Template")

Next click the **Add Template** button.

![Enable Node Template](/assets/images/20190918/node_template_4.png "Enable Node Template")

On the Add Node Template page first select the option for Openstack.  Before filling out the template data we'll need to
retrieve our settings from the Openstack API.

![Enable Node Template](/assets/images/20190918/node_template_2.png "Enable Node Template")

### Obtain Template Settings
The script for obtaining the default API credentials was included in the Openstack bundle we downloaded in 
[Part 2](https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-2/#configure-openstack-bundle).  If they are not already
loaded in your environment change to the openstack-base directory and run the following.
```bash
~$ source openrc
~$ env | grep OS_

OS_USER_DOMAIN_NAME=admin_domain
OS_PASSWORD=Uafe7chee6eigedi
OS_AUTH_URL=http://10.1.20.32:5000/v3
OS_USERNAME=admin
```
>Above, I've only listed the settings needed for the node template.

After loading the API credentials in your environment, you'll need to obtain the default domain and tenant id from the 
Openstack API.
```bash
openstack project show admin -f value -c id
```

Once you have these settings you should have all of the information needed to fill out the node template.  Several of the 
settings shown below were created in [Part 2](https://www.2stacks.net/blog/bare-metal-to-kubernetes-part-2/#validate-deployment)
in order to validate the Openstack installation.  

>Below are all of the required settings with the values relevant to my environment.

| Option | Value |
|--- |--- |
| **authUrl** | http://10.1.20.32:5000/v3 |
| **domainName** | admin_domain |
| **flavorName** | m1.large |
| **floatingipPool** | ext_net |
| **imageName** | bionic |
| **insecure** | enable |
| **keypairName** | os_rsa |
| **netName** | int_net |
| **password** | Uafe7chee6eigedi |
| **privateKeyFile** | <contents of ~/.ssh/os_rsa> |
| **secGroups** | default |
| **sshUser** | ubuntu |
| **tenantid** | ca52ba242b104464b15811d4589f836e |
| **username** | admin |

> Be sure to use tenantId instead of tenantName. Using tenantName will cause docker-machine to use the wrong Openstack API
>version.

Once you've added all of your relevant settings to the template click **Create** to save the node template.

![Enable Node Template](/assets/images/20190918/node_template_3.png "Enable Node Template")

# Add Cluster
Now that we've configured the settings Rancher will use to create nodes within Openstack, we can add our first Kubernetes
cluster.

Click on **Custers** in the UI navigation and then click the **Add Cluster** button.

![Add Cluster](/assets/images/20190918/add_cluster_1.png "Add Cluster")

On the **Add Cluster** page select **Openstack** from the list of infrastructure providers.

![Add Cluster](/assets/images/20190918/add_cluster_2.png "Add Cluster")

Continue configuring the cluster with the following settings;

| Option | Value |
|--- |--- |
| **Cluster Name** | os1 |
| **Name Prefix** | os1-all- |
| **Count** | 3 |
| **Template** | Openstack |

For demonstration purposes select the check boxes next to [etcd], [control plane] and [worker].

[etcd]:https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
[control plane]:https://kubernetes.io/docs/concepts/#kubernetes-control-plane
[worker]:https://kubernetes.io/docs/concepts/architecture/nodes/

![Add Cluster](/assets/images/20190918/add_cluster_3.png "Add Cluster")

These setting will create a three node cluster named **os1** using the previously created Openstack node template.  Each 
node will be configured to run the Kubernetes etcd, control plane and worker services.

>Note: it is recommended to separate the worker nodes from other services in production clusters.

## Configure the Cloud Provider
Before launching the cluster you'll need to configure the Kubernetes [Cloud Provider] for Openstack.

>Cloud Providers enable a set of functionality that integrate with the underlying infrastructure provider, a.k.a the cloud 
>provider. Enabling this integration unlocks a wide set of features for your clusters such as: node address & zone 
>discovery, cloud load balancers for Services with Type=LoadBalancer, IP address management, and cluster networking via 
>VPC routing tables.

[Cloud Provider]:https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/

Under the **Cloud Provider** section select the **Custom** option.  A warning message will appear that states that the 
prerequisites of the underlying cloud provider must be met before enabling this option.  

![Cloud Provider](/assets/images/20190918/cloud_provider_1.png "Cloud Provider")

> All of the prerequisite configurations for Openstack were applied in [Part 2]({% post_url 2019-09-06-bare-metal-to-kubernetes-part-2 %}) of this series.

Also notice the message instructing you to edit the cluster's [YAML] configuration in order to enable the custom
cloud provider.  To begin editing the cluster configuration click the **Edit as YAML** link at the top of the **Cluster
Options** section.

[YAML]:https://yaml.org/

![Cloud Provider](/assets/images/20190918/cloud_provider_2.png "Cloud Provider")

### Obtain Cloud Provider Settings
To configure the Openstack cloud provider we need to add a section of YAML to the top of the configuration.  The YAML
consists of settings similar to those used in the node template.

First we'll need the Openstack API credentials.  As shown [above](#obtain-template-settings) they can be obtained by running.

```bash
~$ source openrc
~$ env | grep OS_

OS_PASSWORD=Uafe7chee6eigedi
OS_AUTH_URL=http://10.1.20.32:5000/v3
OS_USERNAME=admin
```

The remaining configuration settings can be obtained from the Openstack API.  Run the following commands and record their
output.

```bash
openstack project show admin -f value -c id
openstack project show admin -f value -c domain_id
openstack subnet show int_subnet -f value -c id
openstack network show ext_net -f value -c id
openstack router show rtr-01 -f value -c id
```

Now that we have the needed credentials and settings we need to add them to a YAML formatted block of text as shown below.

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

Paste this configuration (substituting your values) in to the top of the Rancher Cloud Provider YAML configuration as 
shown below. 

![Cloud Provider](/assets/images/20190918/cloud_provider_3.png "Cloud Provider")

Now click the **Create** button to launch the Kubernetes cluster.

>For more information on configuring the cloud provider please see the Kubernetes [Cloud Provider] or the [Rancher] documentation.

[Cloud Provider]:https://kubernetes.io/docs/concepts/cluster-administration/cloud-providers/#openstack
[Rancher]:https://rancher.com/docs/rke/latest/en/config-options/cloud-providers/openstack/

## Validate Cluster
As the cluster is deploying we can monitor its progress and status in the Rancher UI by navigating to the **Clusters** page
selecting the link to **os1** under the **Cluster Name** list.  From there click on **Nodes** in the UI navigation.

![Validate Cluster](/assets/images/20190918/validate_cluster_1.png "Validate Cluster")

From here status and error messages will be displayed as the cluster is being deployed.

You can also monitor the cluster build process by connecting to the Rancher instance with SSH.  This is useful for debugging 
any error messages produced by Rancher.

First SSH to the Rancher server.
```bash
ssh -i ~/.ssh/os_rsa ubuntu@<rancher_ip>
```

Next get the rancher Container ID.
```bash
ubuntu@rancher:~$ docker ps
CONTAINER ID        IMAGE                    COMMAND             CREATED             STATUS              PORTS                                      NAMES
21792badbb1c        rancher/rancher:v2.2.8   "entrypoint.sh"     19 hours ago        Up 19 hours         0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp   upbeat_chaplygin
```

Run the following to watch the logs of the rancher container.
```bash
docker logs -f 21792badbb1c
```

You can validate the creation of each cluster node and its associated settings from the Openstack API.
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

Make sure that the status of all hosts created in Openstack is **ACTIVE** and that each is associated with a floating IP 
address.

Once the cluster nodes have been deployed and Rancher has completed the Kubernetes installation process the **State** of 
each node in the Rancher UI should change to **Active**

![Validate Cluster](/assets/images/20190918/validate_cluster_2.png "Validate Cluster")

# Deploy Application
Now that the cluster has been deployed we can validate its functionality by deploying a simple test application.

First navigate to the cluster's default [namespace] by selecting the cluster name (os1) in the top left of the navigation and 
then selecting **Default**.

[namespace]:https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/

![Deploy App](/assets/images/20190918/deploy_app_1.png "Deploy App")

From the [Workloads] view click the **Deploy** button to configure our test application.

[Workloads]:https://rancher.com/docs/rancher/v2.x/en/k8s-in-rancher/workloads/

![Deploy App](/assets/images/20190918/deploy_app_2.png "Deploy App")

## Configure Workload
First set the following name, type and image for the workload.

| Option | Value |
|--- |--- |
| **Name** | webapp |
| **Workload Type** | 3 pods |
| **Docker Image** | [leodotcloud/swiss-army-knife](https://github.com/leodotcloud/swiss-army-knife) |

![Deploy App](/assets/images/20190918/deploy_app_3.png "Deploy App")

Next click the **Add Port** button and configure the port, protocol and [service type].

| Option | Value |
|--- |--- |
| **Container Port** | 80 |
| **Protocol** | TCP |
| **Service Type** | Layer-4 Load Balancer |
| **Listening Port** | 80 |

[service type]:https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types

Next add an environment variables to be used by the application by expanding the **Environment Variables** section and 
clicking the **Add Variable** button.

Add the following environment variable then click the **Launch** button to deploy the application.

| Variable | Value |
|--- |--- |
| **ALPHABET** | A |

![Deploy App](/assets/images/20190918/deploy_app_4.png "Deploy App")

>Adding the environment variable is optional but is useful for testing additional Kubernetes functionality.

# Validate Application
The workload we created will launch three [pods] each with a single instance of the [leodotcloud/swiss-army-knife]
container.  This container contains various tools and a simple web server that can be used to test your Kubernetes 
environment. 

[Pods]:https://kubernetes.io/docs/concepts/workloads/pods/pod/
[leodotcloud/swiss-army-knife]:https://github.com/leodotcloud/swiss-army-knife

## View Application Status
While the workload is is being deployed we can monitor its status and progress from the **Workloads** view in Rancher.

![Validate App](/assets/images/20190918/validate_app_1.png "Validate App")

You can change the default view to **Group By Node** to see the placement of each Pod on the nodes in our Kubernetes 
cluster.

![Validate App](/assets/images/20190918/validate_app_2.png "Validate App")

## View Load Balancer Status
Since we chose a service type of **Layer-4 Load Balancer** Rancher will use the previously configured cloud provider to 
create an instance of the Openstack Octavia load balancer.  It will further configure all of the necessary settings for the
load balancer to direct traffic to our new application.

You can view the deployment status and configuration of the load balancer from the **Load Balancing** view of the Rancher UI.

![Validate LB](/assets/images/20190918/validate_lb_1.png "Validate LB")

>It can take some time for Openstack to create the Octavia virtual machine instance so be patient.

You can validate from the Openstack API that the load balancer instance was created successful by running the following command.

```bash
$ openstack loadbalancer list                             
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| id                                   | name                             | project_id                       | vip_address | provisioning_status | provider |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
| 1301cf7b-9319-4222-a0a0-bcf42fe6d2f0 | ad023b4dbda2e11e98886fa163ea07cc | ca52ba242b104464b15811d4589f836e | 10.0.0.50   | ACTIVE              | amphora  |
+--------------------------------------+----------------------------------+----------------------------------+-------------+---------------------+----------+
```

The **provisioning_status** in the Openstack API should eventually change to **ACTIVE**.  You can further validate that the
**State** shown in the Rancher UI has changed from **Pending** to **Active**.

![Validate LB](/assets/images/20190918/validate_lb_2.png "Validate LB")

## Test Application
Now that the workload and it's load balancer have been deployed we can connect to the web application included in the 
swiss-army-knife container.

From the **Workloads** view of the Rancher UI click the link below the application's name.  This should launch a browser
and connect you to the floating IP address that was assigned to the load balancer.

![Test App](/assets/images/20190918/test_app_1.png "Test App")

The browser window should display the floating IP of the load balancer in the address bar. The page should show the hostname 
and IP address of the container that the load balancer forwarded you to.

![Test App](/assets/images/20190918/test_app_2.png "Test App")

If you refresh the browser periodically you should see that the load balancer forwards you to each of the three containers
that were launched as part of the workload.

![Test App](/assets/images/20190918/test_app_3.png "Test App")

# Conclusion

