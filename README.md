# cdk-nexus

This repository contains documentation for integrating Sonatype Nexus 3 with Canonical Kubernetes (CDK) in order to use it as a private docker registry. 

We cover not only the installation of Canonical Kubernetes using Juju but also the configuration and deployment of Sonatype Nexus 3 as a private docker registry. 

Finally, we cover the post-configuration steps required to configure CDK to use the private registry. 

Nexus is a tried and tested product which has been around for many years and is used by many enterprise businesses world-wide. 

SonaType also provide an open-source, free release of the product (Nexus OSS)  which makes it a perfect pairing with Canonical Kubernetes. 

These steps should also work for Nexus Repository Pro, the commercial supported version of Nexus. 

## Deploying Canonical Kubernetes

Deploying Canonical Kubernetes is really easy, I assume you are running on an Ubuntu machine and you have an AWS account.  Grab a new API key from AWS and put that into:

```
~/.aws/credentials 
```

In this format:

```
[default]
aws_access_key_id=<access_id>
aws_secret_access_key=<secret_key>
```

Next install juju on your machine, we will use the snap:

```
sudo apt-get install snap
snap install juju
```

Next we bootstrap juju so it's ready to use aws:

```
juju bootstrap
```

We follow the steps here for bootstrapping our cloud for AWS: 

```
calvinh@ubuntu-ws:~/.aws$ juju bootstrap
Clouds
aws
aws-china
aws-gov
azure
azure-china
cloudsigma
google
joyent
localhost
oracle
rackspace
salesmaas

Select a cloud [localhost]: aws

Regions in aws:
ap-northeast-1
ap-northeast-2
ap-south-1
ap-southeast-1
ap-southeast-2
ca-central-1
eu-central-1
eu-west-1
eu-west-2
sa-east-1
us-east-1
us-east-2
us-west-1
us-west-2

Select a region in aws [us-east-1]: eu-west-1

Enter a name for the Controller [aws-eu-west-1]: nexus-k8s

Creating Juju controller "nexus-k8s" on aws/eu-west-1
Looking for packaged Juju agent version 2.3.2 for amd64
Launching controller instance(s) on aws/eu-west-1...
 - i-08ce69142f943b5a4 (arch=amd64 mem=4G cores=2)eu-west-1a"
Installing Juju agent on bootstrap instance
Fetching Juju GUI 2.11.3
Waiting for address
Attempting to connect to 172.31.20.176:22
Attempting to connect to 34.244.155.220:22
Connected to 34.244.155.220
Running machine configuration script...


Bootstrap agent now started
Contacting Juju controller at 34.244.155.220 to verify accessibility...
Bootstrap complete, "nexus-k8s" controller now available
Controller machines are in the "controller" model
Initial model "default" added
```

Once the controller has been configured, we now deploy CDK:

```
calvinh@ubuntu-ws:~/.aws$ juju deploy canonical-kubernetes
Located bundle "cs:bundle/canonical-kubernetes-150"
Resolving charm: cs:~containers/easyrsa-27
Resolving charm: cs:~containers/etcd-63
Resolving charm: cs:~containers/flannel-40
Resolving charm: cs:~containers/kubeapi-load-balancer-43
Resolving charm: cs:~containers/kubernetes-master-78
Resolving charm: cs:~containers/kubernetes-worker-81
Executing changes:
- upload charm cs:~containers/easyrsa-27 for series xenial
- deploy application easyrsa on xenial using cs:~containers/easyrsa-27
  added resource easyrsa
- set annotations for easyrsa
- upload charm cs:~containers/etcd-63 for series xenial
- deploy application etcd on xenial using cs:~containers/etcd-63
  added resource etcd
  added resource snapshot
- set annotations for etcd
- upload charm cs:~containers/flannel-40 for series xenial
- deploy application flannel on xenial using cs:~containers/flannel-40
  added resource flannel-amd64
  added resource flannel-s390x
- set annotations for flannel
- upload charm cs:~containers/kubeapi-load-balancer-43 for series xenial
- deploy application kubeapi-load-balancer on xenial using cs:~containers/kubeapi-load-balancer-43
- expose kubeapi-load-balancer
- set annotations for kubeapi-load-balancer
- upload charm cs:~containers/kubernetes-master-78 for series xenial
- deploy application kubernetes-master on xenial using cs:~containers/kubernetes-master-78
  added resource cdk-addons
  added resource kube-apiserver
  added resource kube-controller-manager
  added resource kube-scheduler
  added resource kubectl
- set annotations for kubernetes-master
- upload charm cs:~containers/kubernetes-worker-81 for series xenial
- deploy application kubernetes-worker on xenial using cs:~containers/kubernetes-worker-81
  added resource cni-amd64
  added resource cni-s390x
  added resource kube-proxy
  added resource kubectl
  added resource kubelet
- expose kubernetes-worker
- set annotations for kubernetes-worker
- add relation kubernetes-master:kube-api-endpoint - kubeapi-load-balancer:apiserver
- add relation kubernetes-master:loadbalancer - kubeapi-load-balancer:loadbalancer
- add relation kubernetes-master:kube-control - kubernetes-worker:kube-control
- add relation kubernetes-master:certificates - easyrsa:client
- add relation etcd:certificates - easyrsa:client
- add relation kubernetes-master:etcd - etcd:db
- add relation kubernetes-worker:certificates - easyrsa:client
- add relation kubernetes-worker:kube-api-endpoint - kubeapi-load-balancer:website
- add relation kubeapi-load-balancer:certificates - easyrsa:client
- add relation flannel:etcd - etcd:db
- add relation flannel:cni - kubernetes-master:cni
- add relation flannel:cni - kubernetes-worker:cni
- add unit easyrsa/0 to new machine 0
- add unit etcd/0 to new machine 1
- add unit etcd/1 to new machine 2
- add unit etcd/2 to new machine 3
- add unit kubeapi-load-balancer/0 to new machine 4
- add unit kubernetes-master/0 to new machine 5
- add unit kubernetes-worker/0 to new machine 6
- add unit kubernetes-worker/1 to new machine 7
- add unit kubernetes-worker/2 to new machine 8
Deploy of bundle completed.
```

You can check the deployment status using the following command: 

```
 watch --color juju status --color
```

Note that this will give you the default bundle for CDK which is made up of 9 machines, flannel networking and no RBAC. This is based on the default bundle found here: [https://jujucharms.com/canonical-kubernetes/](https://jujucharms.com/canonical-kubernetes/).

For a more tailored build with Canal or Calico, you can use the bundle builder: [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes). This will generate a bundle file, which is just a big piece of yaml which describes the configuration for the entire cluster, similar to an Ansible Playbook or Puppet Manifest. 

If you have a custom bundle, you would deploy that using a command like this instead:

```
 juju deploy bundle.yaml
```

Eventually the colours will all turn green and your cluster is good to go. To access the cluster, we need to install the kubectl command line client and copy the kubernetes configuration file over for it to use: 

```
 snap install kubectl
```

Next we copy over the configuration file: 

```
  juju scp kubernetes-master/0:/home/ubuntu/config ~/.kube/config
```

Finally, using kubectl we can check that kubernetes cluster interaction is possible: 

```
Kubernetes master is running at https://34.253.164.197:443

Heapster is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kube-dns/proxy
kubernetes-dashboard is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/kubernetes-dashboard/proxy
Grafana is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
InfluxDB is running at https://34.253.164.197:443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy

To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'
```

Now that we have a cluster, we can deploy Nexus on top and configure the cluster to use private registry. 

## Downloading Sonatype Nexus Registry

Sonatype ships Nexus as two different versions: an OSS (Opensource) release and as a Pro release for which Sonatype provides support. 

To download the OSS release, find the download link here: [https://www.sonatype.com/download-oss-sonatype](https://www.sonatype.com/download-oss-sonatype). You should end up with a tar archive with a name like: nexus-3.8.0-02-unix.tar.gz. 


## Deploying Sonatype Nexus Registry

Nexus is distributed as a tar archive or docker container. This means we can install it on a Virtual Machine, Physical Machine or as an LXD Container.  
We will start as with the tar archive, make sure you download it using the previous instructions. We will extract this somewhere useful, the manual suggests using /opt/:

```
# Create /opt/nexus and extract nexus, set user and file permissions 
sudo useradd nexus --no-create-home --disabled-login --disabled-password nexus
sudo mkdir /opt/nexus
sudo tar xvf ~/Downloads/nexus*.tar.gz -o /opt/nexus/
chown nexus:nexus -R /opt/nexus
``` 

Once we've extracted Nexus, we now need to install Java for Nexus to work. According to the documentation, we should use Oracle Java: 

```

```

## Deploying Sonatype Nexus Registry as a Container on Kubernetes
## Configuring Nexus Registry as Private Docker Registry
## Configuring Nexus Registry as a Proxy Docker Registry
## Configuring Nexus as a Repository Group

## Using the Private Registry

To push or pull Docker Container from/to the Private Registry, the Docker daemon running on the Kubernetes worker nodes needs to be configured to use it. 

There are two things which need to be configured:

- If the Nexus Private Registry is using self signed certificates, we must tell the Docker Daemon to ignore the SSL/TLS verification checks on the certicate for our Registry. 
- We must log into the Registry on the kubernetes workers or provide the credentials to the Docker daemon through the runtime parameters. 

To push containers to the Private Registry, 

To pull containers from the Private Registry, 

## Getting Support & Help

To open issues with CDK, please open an issue with the repository here [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes). If you want professional support for Kubernetes please contact Canonical Sales [https://www.ubuntu.com/kubernetes](https://www.ubuntu.com/kubernetes). 

To get help with Nexus, you can open tickets on their Jira here [https://issues.sonatype.org/browse/NEXUS](https://issues.sonatype.org/browse/NEXUS). You can also try looking at the posts on Stack Overflow tagged with Nexus 3 [http://stackoverflow.com/questions/tagged/nexus3](http://stackoverflow.com/questions/tagged/nexus3) and the Nexus Repository User List [https://groups.google.com/a/glists.sonatype.com/forum/?hl=en#!forum/nexus-users](https://groups.google.com/a/glists.sonatype.com/forum/?hl=en#!forum/nexus-users). 

Finally, Sonatype also sell support for Nexus Pro, the commercial version of Nexus which can be found here [https://www.sonatype.com/repository-pro-3-upgrade](https://www.sonatype.com/repository-pro-3-upgrade). 

## Summary 

We have looked at deploying Canonical Kubernetes with Sonatype Nexus in various configurations. We've covered how to configure Kubernetes to use the private registry and how to push or pull containers into it.  

## Useful Links

- [https://www.ubuntu.com/kubernetes](https://www.ubuntu.com/kubernetes)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes)
- [https://github.com/sonatype/nexus-public](https://github.com/sonatype/nexus-public)
- [https://www.sonatype.com/download-oss-sonatype](https://www.sonatype.com/download-oss-sonatype)
- [https://www.sonatype.com/nexus-repository-oss](https://www.sonatype.com/nexus-repository-oss)
- [https://www.sonatype.com/download-nexus-repository-trial](https://www.sonatype.com/download-nexus-repository-trial)
- [https://books.sonatype.com/nexus-book/3.0/reference/docker.html](https://books.sonatype.com/nexus-book/3.0/reference/docker.html)
- [https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/](https://kubernetes.io/docs/getting-started-guides/ubuntu/installation/)
- [https://github.com/juju-solutions/bundle-canonical-kubernetes](https://github.com/juju-solutions/bundle-canonical-kubernetes)

