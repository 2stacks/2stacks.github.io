---
title: "Rancher 2 and Letsencrypt"
excerpt: "How to automate Letsencrypt certificates with Rancher 2 Ingress"
layout: single
toc: true
toc_sticky: true
categories:
  - Blog
tags:
  - Kubernetes
  - Rancher
  - Letsencrypt
---

I decided to write this post to help with the [discussion on the Rancher Forum][0] regarding the difficulties many were having
trying to setup Letsencrypt certificates with cert-manager.  I've borrowed and owe credit to work that's already been
documented [here][1] and I'll try to stick to the steps I took to enable the full automation of the certificate process.

> NOTE: This post has been updated to demonstrate usage of a newer version of cert-manager provided by JetPack. Notices
> from Letsencrypt have been sent regarding the blocking of versions older than v0.8.0.

Prerequisites
-------------

I'm going to assume a few things for brevity.

* You have a functioning Rancher 2.0 Cluster
* You have kubectl set up with your [Rancher Kubeconfig File][2]
* You have a publicly reachable dns service that points the domain, for which you want to issue certificates, to your cluster nodes, loadbalancer or port forwarder if using NAT.
* If your cluster is behind NAT you have set up split DNS. (See section on DNS)

Installation
------------
### Enable JetPack Helm Repository

The default library repository in Rancher only includes cert-manager versions up to v0.5.2. The first thing we need to
do is add the JetPack repository to Rancher.

In the Rancher UI navigation go to **Tools** and select **Catalogs**.

![Add Catalog](/assets/images/20190524/Selection_001.png "Add Catalog")

Next click the **Add Catalog** button.

![Add Catalog](/assets/images/20190524/Selection_002.png "Add Catalog")

Give the new catalog a name like `jetstack` and configure `https://charts.jetstack.io` as the catalog URL.

![Add Catalog](/assets/images/20190524/Selection_003.png "Add Catalog")

### Install cert-manager

Lets start by ensuring we are working from a clean slate.  If you have attempted to install cert-manager before remove any
existing resources and verify that the required name space isn't in use.

```bash
~$ kubectl get all -n cert-manager
No resources found.
```
```bash
~$ kubectl describe clusterissuers letsencrypt-staging
error: the server doesnt have a resource type "clusterissuers"
```

Before installing the cert-manager application in Rancher we need to first add the [Customer Resource Definition]. From
your workstation execute the following.

```bash
kubectl apply -f https://raw.githubusercontent.com/jetstack/cert-manager/release-0.9/deploy/manifests/00-crds.yaml
```

Now label the kube-system namespace to disable resource validation.

```bash
kubectl label namespace kube-system certmanager.k8s.io/disable-validation=true
```
 
Next navigate to the **Apps** section of the Rancher **System** Project.

![alt text](/assets/images/20190524/nav_to_apps.png "Navigate to Apps")

Next click the **Launch** button and type **cert** in the search menu. Click on the cert-manager provided by the
JetStack catalog.

![alt text](/assets/images/20190524/Selection_009.png "Launch cert-manager")

Change the namespace to **kube-system** and set the template version to **v0.9.1**. 

> Previous versions of cert-manager used the namespace 'cert-manager'. Make sure you deploy to kube-system or the
> installation will fail.

![alt text](/assets/images/20190524/Selection_010.png "Configure cert-manager")

### Verify Installation

Now verify your app deployment with kubectl.

```bash
~$ kubectl get all -n kube-system | grep cert-manager
pod/cert-manager-5b9ff77b7-lhsb4             1/1     Running     0          165m
pod/cert-manager-cainjector-59d69b9b-9f5ng   1/1     Running     0          165m
pod/cert-manager-webhook-cfd6587ff-fz2cv     1/1     Running     0          125m
service/cert-manager-webhook   ClusterIP   10.43.104.116   <none>        443/TCP                  165m
deployment.apps/cert-manager              1/1     1            1           165m
deployment.apps/cert-manager-cainjector   1/1     1            1           165m
deployment.apps/cert-manager-webhook      1/1     1            1           165m
replicaset.apps/cert-manager-5b9ff77b7             1         1         1       165m
replicaset.apps/cert-manager-cainjector-59d69b9b   1         1         1       165m
replicaset.apps/cert-manager-webhook-cfd6587ff     1         1         1       165m
```

Unlike previous versions of the Rancher cert-manager application, you'll need to create your own [Cluster Issuer]. First
create a file similar to the following.
```bash
~$ cat cluster-issuer.yaml 
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    # The ACME server URL
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    # Email address used for ACME registration
    email: 2stacks@2stacks.net
    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    # Enable HTTP01 validations
    http01: {}
```

Now apply the configuration with;

```bash
kubectl create -f cluster-issuer.yaml
```

Verify the creation of the Cluster Issuer with;

```bash
kubectl describe clusterissuers letsencrypt-
```

The important thing to note is that the cert-manager pod is running and that your email account was successfully
registered with the ACME server API. If your output isn't similar to above check the logs of the cert-manager pod for
any issues.

![alt text](/assets/images/20190524/view_logs.png "View Logs")

Notice in the logs below the message that says `Not syncing ingress default/nginx as it does not contain necessary annotations`.
I precreated a test nginx deployment which we are going to use to test the `ingress-shim` functionality of cert-manager.

![alt text](/assets/images/20190524/logs.png "Logs")

Configuring Ingress
-------------------
### Deploy a Test Workload

I chose to deploy an nginx container as a test since it provides the default server and nginx welcome page without any
configuration.

From the `Workloads` section of your chosen Rancher Project click the `Deploy` button.  Give the workload a name, choose
the nginx `Docker Image` of your choice, leave the NameSpace set to default.  Add a `Port Mapping` for port 80 and publish
the service as a `Cluster IP (Internal only)`.  Click `Launch` to create the new workload.

![alt text](/assets/images/20190524/new_workload.png "Launch Workload")

### Create an Ingress

Next from the `Load Balancing` menu, click the `Add Ingress` button.

In the `Add Ingress` configuration page, give the Ingress a name and leave the Namespace set to default.  Under the `Rules`
section select the option for `Specify a hostname to use`.  My test lab is setup to use the domain bsptn.xyz so I have 
configured the `Request Host` name as "nginx.bsptn.xyz"

By default Rancher chooses a `Workload` as the default for the `Target Backend`.  We want to use the service that was
automatically created when we deployed our nginx workload.  Click the minus sign button to the right of the `Port` field
to remove the existing Target Backend.

![alt text](/assets/images/20190524/configure_ingress.png "Configure Ingress")

Now click the `Service` button next to `Target Backend`. Set the `Path` to "/" and in the `Target` drop down select the nginx
service.

![alt text](/assets/images/20190524/target_backend.png "Target Backend")

At this point you should save the ingress without configuring any SSL Certificates or Annotations.  You should verify that
you nginx deployment is reachable via the ingress from both the Internet and your internal network.  If you can not then
you have more work to do with DNS, Load Balancing, NAT etc. before you can proceed to the next step.

![alt text](/assets/images/20190524/nginx_http.png "Nginx Http")

### Edit the Ingress

At this point if you have verified that your ingress service is reachable you can proceed to adding the annotations required
to automatically request and deploy a certificate.  From the `Load Balancing` menu click the drop down to the far right of
the nginx ingress and then select `edit`

![alt text](/assets/images/20190524/edit_ingress.png "Edit Ingress")

Scroll to the bottom of the page and expand the `SSL/TLS Certificates` and `Labels & Annotations`  sections.  First click
the `Add Certificate` button under the SSL/TLS section.  Leave the option set for `Use default ingress controller certificate`.
Don't worry we will manually edit this in the Yaml later.  Set the `Host` section to the FQDN of your service.

![alt text](/assets/images/20190524/add_cert.png "Add Certificate")

Now you need to add the annotations as per the [ingress-shim][4] documentation.  If you forget the required annotations you
can view the docs.  They are also provided in the `Notes` section when you first launched the cert-manager app.

![alt text](/assets/images/20190524/shim_settings.png "Ingress-shim")

Under the `Labels & Annotations` section click the `Add Annotation` button twice and add the following annotations.
* `kubernetes.io/tls-acme: "true"`
* `certmanager.k8s.io/cluster-issuer: letsencrypt-staging`

Now click `Save` to update the Ingress.

![alt text](/assets/images/20190524/save_ingress.png "Save Ingress Changes")

The final step can not be performed through the Rancher Gui at this time.  We'll need to manually edit the Yaml of the Ingress
we just created.

From the `Load Balancing` menu click the drop down to the far right of the nginx ingress and then select `View/Edit YAML`.

![alt text](/assets/images/20190524/edit_yaml.png "Edit YAML")

Scroll the bottom of the Yaml config and under `spec -> tls -> hosts` add the `secretName` definition with the resource 
name you want the certificate to be saved with.  I've chosen `nginx-bsptn-xyz-crt` for my implementation.  Now click the
save button.

![alt text](/assets/images/20190524/add_secretname.png "Add SecretName")

If everything to this point has gone well and if you watch carefully, cert-manager will temporarily create a new Ingress
for the purposes of performing HTTP01 challenge verification.

![alt text](/assets/images/20190524/challenge_ingress.png "Challenge Ingress")

If you missed it or if something went wrong now is a good time to review the cert-manager logs.  I've included my logs
from the last time the ingress-shim failed due to missing configurations up until the certificate is pulled and the temporary
HTTP01 challenge Ingress is removed.

```bash
E0530 23:01:37.571902 1 controller.go:177] ingress-shim controller: Re-queuing item "default/nginx" due to error processing: TLS entry 0 for ingress "nginx" must specify a secretName
I0530 23:02:37.572406 1 controller.go:168] ingress-shim controller: syncing item 'default/nginx'
I0530 23:02:37.598364 1 controller.go:182] ingress-shim controller: Finished processing work item "default/nginx"
I0530 23:02:37.598406 1 controller.go:168] ingress-shim controller: syncing item 'default/nginx'
I0530 23:02:37.598430 1 sync.go:140] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" already exists
I0530 23:02:37.598462 1 sync.go:143] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" is up to date
I0530 23:02:37.598480 1 controller.go:182] ingress-shim controller: Finished processing work item "default/nginx"
I0530 23:02:39.598243 1 controller.go:171] certificates controller: syncing item 'default/nginx-bsptn-xyz-crt'
I0530 23:02:39.598704 1 sync.go:274] Preparing certificate default/nginx-bsptn-xyz-crt with issuer
I0530 23:02:39.599901 1 prepare.go:263] Cleaning up previous order for certificate default/nginx-bsptn-xyz-crt
I0530 23:02:39.599939 1 prepare.go:279] Cleaning up old/expired challenges for Certificate default/nginx-bsptn-xyz-crt
I0530 23:02:39.599998 1 logger.go:38] Calling CreateOrder
I0530 23:02:40.148955 1 acme.go:126] Created order for domains: [{dns nginx.bsptn.xyz}]
I0530 23:02:40.149074 1 logger.go:73] Calling GetAuthorization
I0530 23:02:40.212749 1 logger.go:93] Calling HTTP01ChallengeResponse
I0530 23:02:40.212917 1 prepare.go:279] Cleaning up old/expired challenges for Certificate default/nginx-bsptn-xyz-crt
I0530 23:02:40.212949 1 logger.go:68] Calling GetChallenge
I0530 23:02:40.340693 1 pod.go:65] No existing HTTP01 challenge solver pod found for Certificate "default/nginx-bsptn-xyz-crt". One will be created.
I0530 23:02:40.394210 1 service.go:51] No existing HTTP01 challenge solver service found for Certificate "default/nginx-bsptn-xyz-crt". One will be created.
I0530 23:02:40.506689 1 ingress.go:49] Looking up Ingresses for selector certmanager.k8s.io/acme-http-domain=806880787,certmanager.k8s.io/acme-http-token=1172442703
I0530 23:02:40.506761 1 ingress.go:102] No existing HTTP01 challenge solver ingress found for Certificate "default/nginx-bsptn-xyz-crt". One will be created.
I0530 23:02:40.595694 1 helpers.go:194] Setting lastTransitionTime for Certificate "nginx-bsptn-xyz-crt" condition "Ready" to 2019-05-30 23:02:40.595666151 +0000 UTC m=+8461.551329830
I0530 23:02:40.595767 1 sync.go:276] Error preparing issuer for certificate default/nginx-bsptn-xyz-crt: http-01 self check failed for domain "nginx.bsptn.xyz"
E0530 23:02:40.595863 1 sync.go:197] [default/nginx-bsptn-xyz-crt] Error getting certificate 'nginx-bsptn-xyz-crt': secret "nginx-bsptn-xyz-crt" not found
E0530 23:02:40.711924 1 controller.go:180] certificates controller: Re-queuing item "default/nginx-bsptn-xyz-crt" due to error processing: http-01 self check failed for domain "nginx.bsptn.xyz"
I0530 23:02:40.721555 1 controller.go:168] ingress-shim controller: syncing item 'default/nginx'
I0530 23:02:40.723070 1 sync.go:140] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" already exists
I0530 23:02:40.723508 1 sync.go:143] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" is up to date
I0530 23:02:40.723762 1 controller.go:182] ingress-shim controller: Finished processing work item "default/nginx"
I0530 23:02:44.713550 1 controller.go:171] certificates controller: syncing item 'default/nginx-bsptn-xyz-crt'
I0530 23:02:44.713701 1 sync.go:274] Preparing certificate default/nginx-bsptn-xyz-crt with issuer
I0530 23:02:44.714024 1 logger.go:43] Calling GetOrder
I0530 23:02:44.808517 1 logger.go:73] Calling GetAuthorization
I0530 23:02:44.897577 1 logger.go:93] Calling HTTP01ChallengeResponse
I0530 23:02:44.897678 1 prepare.go:279] Cleaning up old/expired challenges for Certificate default/nginx-bsptn-xyz-crt
I0530 23:02:44.897711 1 logger.go:68] Calling GetChallenge
I0530 23:02:44.973655 1 http.go:134] wrong status code '503'
I0530 23:02:44.974074 1 ingress.go:49] Looking up Ingresses for selector certmanager.k8s.io/acme-http-domain=806880787,certmanager.k8s.io/acme-http-token=1172442703
I0530 23:02:44.974202 1 helpers.go:201] Found status change for Certificate "nginx-bsptn-xyz-crt" condition "Ready": "False" -> "False"; setting lastTransitionTime to 2019-05-30 23:02:44.974192204 +0000 UTC m=+8465.929855813
I0530 23:02:44.974303 1 sync.go:276] Error preparing issuer for certificate default/nginx-bsptn-xyz-crt: http-01 self check failed for domain "nginx.bsptn.xyz"
E0530 23:02:44.974380 1 sync.go:197] [default/nginx-bsptn-xyz-crt] Error getting certificate 'nginx-bsptn-xyz-crt': secret "nginx-bsptn-xyz-crt" not found
I0530 23:02:45.004974 1 controller.go:168] ingress-shim controller: syncing item 'default/nginx'
I0530 23:02:45.005147 1 sync.go:140] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" already exists
I0530 23:02:45.005183 1 sync.go:143] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" is up to date
I0530 23:02:45.005248 1 controller.go:182] ingress-shim controller: Finished processing work item "default/nginx"
E0530 23:02:45.008794 1 controller.go:180] certificates controller: Re-queuing item "default/nginx-bsptn-xyz-crt" due to error processing: http-01 self check failed for domain "nginx.bsptn.xyz"
I0530 23:02:45.599202 1 controller.go:168] ingress-shim controller: syncing item 'default/cm-acme-http-solver-wtcbm'
I0530 23:02:45.599414 1 sync.go:65] Not syncing ingress default/cm-acme-http-solver-wtcbm as it does not contain necessary annotations
I0530 23:02:45.599638 1 controller.go:182] ingress-shim controller: Finished processing work item "default/cm-acme-http-solver-wtcbm"
I0530 23:03:01.005455 1 controller.go:171] certificates controller: syncing item 'default/nginx-bsptn-xyz-crt'
I0530 23:03:01.006291 1 sync.go:274] Preparing certificate default/nginx-bsptn-xyz-crt with issuer
I0530 23:03:01.007303 1 logger.go:43] Calling GetOrder
I0530 23:03:01.187043 1 logger.go:73] Calling GetAuthorization
I0530 23:03:01.271914 1 logger.go:93] Calling HTTP01ChallengeResponse
I0530 23:03:01.272196 1 prepare.go:279] Cleaning up old/expired challenges for Certificate default/nginx-bsptn-xyz-crt
I0530 23:03:01.272583 1 logger.go:68] Calling GetChallenge
I0530 23:03:11.760008 1 prepare.go:488] Accepting challenge for domain "nginx.bsptn.xyz"
I0530 23:03:11.760125 1 logger.go:63] Calling AcceptChallenge
I0530 23:03:12.179202 1 prepare.go:500] Waiting for authorization for domain "nginx.bsptn.xyz"
I0530 23:03:12.179361 1 logger.go:78] Calling WaitAuthorization
I0530 23:03:14.383630 1 prepare.go:510] Successfully authorized domain "nginx.bsptn.xyz"
I0530 23:03:14.383916 1 prepare.go:303] Cleaning up challenge for domain "nginx.bsptn.xyz" as part of Certificate default/nginx-bsptn-xyz-crt
I0530 23:03:14.642912 1 ingress.go:49] Looking up Ingresses for selector certmanager.k8s.io/acme-http-domain=806880787,certmanager.k8s.io/acme-http-token=1172442703
I0530 23:03:14.674932 1 sync.go:281] Issuing certificate...
I0530 23:03:14.675267 1 logger.go:43] Calling GetOrder
I0530 23:03:15.898507 1 logger.go:58] Calling FinalizeOrder
I0530 23:03:16.872750 1 issue.go:196] successfully obtained certificate: cn="nginx.bsptn.xyz" altNames=[nginx.bsptn.xyz] url="https://acme-staging-v02.api.letsencrypt.org/acme/order/9448784/35891936"
I0530 23:03:16.931122 1 sync.go:300] Certificate issued successfully
I0530 23:03:16.931385 1 helpers.go:201] Found status change for Certificate "nginx-bsptn-xyz-crt" condition "Ready": "False" -> "True"; setting lastTransitionTime to 2019-05-30 23:03:16.931293187 +0000 UTC m=+8497.886956711
I0530 23:03:16.932560 1 sync.go:206] Certificate default/nginx-bsptn-xyz-crt scheduled for renewal in 1438 hours
I0530 23:03:16.951754 1 controller.go:185] certificates controller: Finished processing work item "default/nginx-bsptn-xyz-crt"
I0530 23:03:16.952862 1 controller.go:168] ingress-shim controller: syncing item 'default/nginx'
I0530 23:03:16.953088 1 sync.go:140] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" already exists
I0530 23:03:16.953906 1 sync.go:143] Certificate "nginx-bsptn-xyz-crt" for ingress "nginx" is up to date
I0530 23:03:16.954138 1 controller.go:182] ingress-shim controller: Finished processing work item "default/nginx"
I0530 23:03:18.953126 1 controller.go:171] certificates controller: syncing item 'default/nginx-bsptn-xyz-crt'
I0530 23:03:18.954881 1 sync.go:206] Certificate default/nginx-bsptn-xyz-crt scheduled for renewal in 1438 hours
I0530 23:03:18.955045 1 controller.go:185] certificates controller: Finished processing work item "default/nginx-bsptn-xyz-crt"
I0530 23:03:19.674777 1 controller.go:168] ingress-shim controller: syncing item 'default/cm-acme-http-solver-wtcbm'
E0530 23:03:19.674894 1 controller.go:198] ingress 'default/cm-acme-http-solver-wtcbm' in work queue no longer exists
I0530 23:03:19.674942 1 controller.go:182] ingress-shim controller: Finished processing work item "default/cm-acme-http-solver-wtcbm"
```
<br/>

Verification
------------
If you get a log message similar to `issue.go:196] successfully obtained certificate: cn="nginx.bsptn.xyz"` chances are
you are in good shape.  I'll run through just a couple of things to verify everything is working.

Within the project in which you created your Ingress navigate to `Resources` -> `Certificates` and verify that your certificate
resource has been created in the cluster.

![alt text](/assets/images/20190524/verify_certificate.png "Verify Certificate Resource")

The Rancher UI does't give a lot of information about the certificate so to view it's details we'll need to use Kubectl.

```bash
~$ kubectl get certificates
NAME                  AGE
nginx-bsptn-xyz-crt   26m
```
```bash
~$ kubectl describe certificates nginx-bsptn-xyz-crt 
Name:         nginx-bsptn-xyz-crt
Namespace:    default
Labels:       <none>
Annotations:  <none>
API Version:  certmanager.k8s.io/v1alpha1
Kind:         Certificate
Metadata:
  Creation Timestamp:  2019-05-30T23:02:37Z
  Generation:          4
  Owner References:
    API Version:           extensions/v1beta1
    Block Owner Deletion:  true
    Controller:            true
    Kind:                  Ingress
    Name:                  nginx
    UID:                   <some_uid>
  Resource Version:        1023500
  Self Link:               /apis/certmanager.k8s.io/v1alpha1/namespaces/default/certificates/nginx-bsptn-xyz-crt
  UID:                     <some_uid>
Spec:
  Acme:
    Config:
      Domains:
        nginx.bsptn.xyz
      Http 01:
        Ingress:  
  Dns Names:
    nginx.bsptn.xyz
  Issuer Ref:
    Kind:       ClusterIssuer
    Name:       letsencrypt-staging
  Secret Name:  nginx-bsptn-xyz-crt
Status:
  Acme:
    Order:
      URL:  https://acme-staging-v02.api.letsencrypt.org/acme/order/<number>/<number>
  Conditions:
    Last Transition Time:  2019-05-30T23:03:16Z
    Message:               Certificate issued successfully
    Reason:                CertIssued
    Status:                True
    Type:                  Ready
    Last Transition Time:  <nil>
    Message:               Order validated
    Reason:                OrderValidated
    Status:                False
    Type:                  ValidateFailed
Events:
  Type    Reason          Age   From          Message
  ----    ------          ----  ----          -------
  Normal  CreateOrder     26m   cert-manager  Created new ACME order, attempting validation...
  Normal  DomainVerified  25m   cert-manager  Domain "nginx.bsptn.xyz" verified with "http-01" validation
  Normal  IssueCert       25m   cert-manager  Issuing certificate...
  Normal  CertObtained    25m   cert-manager  Obtained certificate from ACME server
  Normal  CertIssued      25m   cert-manager  Certificate issued successfully
```
<br/>

Again, if everything worked under the `Events` section you should be able to see that the certificate was issued successfully.

The last obovious verification is to load the nginx web page and verify with the a browser that a LetsEncypt certificate has
been issued from the staging API.  Since I chose to issue certs from the staging API the browser will still generate a certificate
error however, you can see from the certificate details that is has been Issued By `Fake LE Intermediate X1`.

![alt text](/assets/images/20190524/verify_browser.png "Verify Browser")

Additional Notes
----------------
I'm sure if this process works for you you'll want to proceed to issuing certs from LetsEncrypt's production API.  I have
test this exact same procedure using the `letsencrypt-prod` cluster issuer on a clean cluster.  I have not attempted to run
more than one cluster issue on the same Rancher cluster.  If you're ready to issue valid certificates I recommend you delete
the cert-manager app you deployed and start over.  You should be able to follow all of the steps in this post replacing all
instances have `letsencrypt-staging` with `letsencrypt-prod`.

### DNS

You may or may not have noticed that may cluster nodes have private IP addresses.  I'll share a little about my setup in case
some of you are attempting to use cert-manager on a private lab network.  I assume that clusters deployed in public clouds
won't have as many http01 verification issues but I could be wrong.

* First, I have a wildcard dns record in AWS Route53 that points *.bsptn.xyz to a device performing NAT for my lab environment.
* That NAT boundary forwards all port 80 and 443 to an L4-7 loadbalancer that services my Kubernetes clusters.
* I have a private DNS server built with [PowerDNS][5] for internal name resolution of private IPs.  I chose PowerDNS because
it provides an API that integrates with the Kubernetes add on service [external-dns][6]
* Inside my Kubernetes cluster's I deploy the Bitnami version of the external-dns application.

Any Ingress I create in my clusters is automatically registered in PowerDNS via the external-dns application.  This make the
process of performing HTTP01 verification much easier in my environment.

If you're interested in more details of how I set up my lab environment feel free to contact me.  I have posted a lot of the work
I've done to GitHub [@2stacks][7] and I mostly use Terraform so that my deployments are repeatable.


[0]: https://forums.rancher.com/t/rancher-2-and-letsencrypt/10492
[1]: https://www.idealcoders.com/posts/rancher/2018/06/rancher-2-x-and-lets-encrypt-with-cert-manager-and-nginx-ingress/
[2]: https://rancher.com/docs/rancher/v2.x/en/cluster-admin/kubectl/#accessing-clusters-with-kubectl-and-a-kubeconfig-file
[3]: https://rancher.com/docs/rancher/v2.x/en/catalog/built-in/
[4]: https://cert-manager.readthedocs.io/en/latest/tasks/issuing-certificates/ingress-shim.html#supported-annotations
[5]: https://www.powerdns.com/
[6]: https://github.com/kubernetes-incubator/external-dns
[7]: https://github.com/2stacks
[Customer Resource Definition]:https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/#customresourcedefinitions
[Cluster Issuer]:https://docs.cert-manager.io/en/release-0.9/reference/clusterissuers.html