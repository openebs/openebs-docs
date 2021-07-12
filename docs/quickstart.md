---
id: quickstart
title: OpenEBS Quickstart Guide
sidebar_label: Quickstart
---
------

<br>
This guide will help you to setup OpenEBS and use OpenEBS Volumes to run your Kubernetes Stateful Workloads. If you are new to running Stateful workloads in Kubernetes, you will need to familiarize yourself with [Kubernetes Storage Concepts](/docs/next/k8s-storage.html). 


In most cases, the following steps is all you need to install OpenEBS. You can read through the rest of the document to understand the choices you have and optimize OpenEBS for your Kubernetes cluster. 
 
:::tip QUICKSTART
Install using helm
```
helm repo add openebs https://openebs.github.io/charts
helm repo update
helm install openebs --namespace openebs openebs/openebs --create-namespace
```

Install using kubectl 
```
kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
```
:::

## How to setup and use OpenEBS?

OpenEBS seamlessly integrates into the overall workflow tooling that Kubernetes administrators and users have around Kubernetes. 

The OpenEBS workflow fits nicely into the reconcilation pattern introduced by Kubernetes, paving the way for a Declarative Storage Control Plane as shown below: 

<br>
<a href="/docs/assets/control-plane-overview.svg" target="_blank"><img src="/docs/assets/control-plane-overview.svg" alt="drawing" width="80%"/></a>
<br>


### 1. Kubernetes Cluster Design

As a Kubernetes cluster administrator, you will have to work with your Platform or Infrastructure teams on the composition of the Kubernetes worker nodes like - RAM, CPU, Network and the storage devices attached to the worker nodes. The [resources available to the Kubernetes nodes](docs/next/casengines.html#node-capabilities) determine what OpenEBS engines to use for your stateful workloads. 

As a Kubernetes cluster administrator or Platform SREs you will have to decide which deployment strategy works best for you - either use an hyperconverged mode where Stateful applications and storage volumes are co-located or run Stateful applications and storage on different pools of nodes. 

For installing OpenEBS, you Kubernetes cluster should meet the following:
- Kubernetes 1.18 or newer is recommended. 
- Based on the selected data engine, the nodes should be prepared with additional packages like:
  - Installing the ext4, xfs, nfs, lvm, zfs or iscsi, nvme packages.
  - Prepare the devices for use by data engines like - making sure there are no the filesystem installed or by creating an LVM volume group or ZFS Pool or partition the drives if required. 
- Based on whether you are using a upstream Kubernetes cluster or using a managed Kubernetes cluster like AKS, Rancher, OpenShift, GKE, there may be additional steps required. 

Please read through the relevant section of the [pre-requisites](/docs/next/prerequisites.html) for your Kubernetes platform, Operating System of the worker nodes.
- [Ubuntu](/docs/next/prerequisites.html#ubuntu)
- [RHEL](/docs/next/prerequisites.html#rhel)
- [CentOS](/docs/next/prerequisites.html#centos)
- [OpenShift](/docs/next/prerequisites.html#openshift)
- [Rancher](/docs/next/prerequisites.html#rancher)
- [ICP](/docs/next/prerequisites.html#icp)
- [EKS](/docs/next/prerequisites.html#eks)
- [GKE](/docs/next/prerequisites.html#gke)
- [AKS](/docs/next/prerequisites.html#aks)
- [Digital Ocean](/docs/next/prerequisites.html#do)
- [Konvoy](/docs/next/prerequisites.html#konvoy)

If your platform is missing in the above list, please [raise an issue on the docs](https://github.com/openebs/openebs/issues/new/choose) or reach us on the [community slack](/docs/next/support.html) to let us know. 

### 2. Install OpenEBS and Setup Storage Classes

OpenEBS is Kubernetes native, which makes it possible to install OpenEBS into your Kubernetes cluster - just like any other application. 

You can install OpenEBS only using Kubernetes admin context as you will require cluster level permissions to create Storage Classes. 

OpenEBS offers different modes of [installation](/docs/next/installation.html). The most popular ones are using:
- [OpenEBS Helm chart](/docs/next/installation.html#installation-through-helm)
- [OpenEBS YAML(s) via `kubectl`](/docs/next/installation.html#installation-through-kubectl)

OpenEBS will install a couple of default storage classes that you an use for Local Volumes (`openebs-hostpath`) and Replicated Volumes (`openebs-hostpath`). The data of the volumes created by these default storage classes will be saved under `/var/openebs`. 

As a Platform SRE / Cluster Administrator, you can customize several things about OpenEBS installer to suite your specific environment and create the setup the required Storage Classes. You can jump to the relevant sections based on your choice of [data engines](docs/next/casengines.html#data-engine-capabilities):

- [Local PV hostpath](/docs/next/uglocalpv-hostpath.html)
- [Local PV device](/docs/next/uglocalpv-device.html)
- [Local PV ZFS](https://github.com/openebs/zfs-localpv)
- [Local PV LVM](https://github.com/openebs/lvm-localpv)
- [Local PV Rawfile](https://github.com/openebs/rawfile-localpv)
- [Replicated PV Jiva](https://github.com/openebs/jiva-operator)
- [Replicated PV cStor](https://github.com/openebs/cstor-operators/blob/master/docs/quick.md)
- [Replicated PV Mayastor](/docs/next/ugmayastor.html)

### 3. Deploy Stateful Workloads

The application developers will launch their application (stateful workloads) that will in turn create Persistent Volume Claims for requesting the Storage or Volumes for their pods. The Platform teams can provide templates for the applications with associated PVCs or application developers can select from the list of storage classes available for them. 

As an application developer all you have to do is substitute the `StorageClass` in your PVCs with the OpenEBS Storage Classes available in your Kubernetes cluster. 

Here are examples of some applications using OpenEBS: 
- <a href="/docs/next/mysql.html" target="_blank"> MySQL </a>
- <a href="/docs/next/postgres.html" target="_blank"> PostgreSQL </a>
- <a href="/docs/next/percona.html" target="_blank"> Percona </a>
- <a href="/docs/next/redis.html" target="_blank"> Redis </a>
- <a href="/docs/next/mongo.html" target="_blank"> MongoDB </a>
- <a href="/docs/next/cassandra.html" target="_blank"> Cassandra </a>
- <a href="/docs/next/prometheus.html" target="_blank"> Prometheus </a>
- <a href="/docs/next/elasticsearch.html" target="_blank"> Elastic </a>
- <a href="/docs/next/minio.html" target="_blank"> Minio </a>
- <a href="/docs/next/rwm.html" target="_blank"> Wordpress using NFS </a>


### 4. Dynamic Persistent Volume Provisioning

The Kubernetes CSI (provisioning layer) will intercept the requests for the Persistent Volumes and forward the requests to the OpenEBS Control Plane components to service the requests. The information provided in the StorageClass combined with requests from PVCs will determine the right OpenEBS control component to receive the request. 

OpenEBS control plane will then process the request and create the Persistent Volumes using the specified local or replicated engines. The data engine services like target and replica are deployed as Kubernetes applications as well. The containers provide storage for the containers. The new containers launched for serving the applications will be available in the `openebs` namespace. 

With the magic of OpenEBS and Kubernetes, the volumes should be provisioned, pods scheduled and application ready to serve. For this magic to happen, the prerequisites should be met. Check out our [troubleshooting section](docs/next/troubleshooting.html) for some of the common errors that users run into due to setup issues. 


### 5. Managing the Life cycle of OpenEBS components

Once the workloads are up and running, the platform or the operations team can observe the system using the cloud native tools like Prometheus, Grafana and so forth. The operational tasks are a shared responsibility across the teams: 
* Application teams can watch out for the capacity and performance and tune the PVCs accordingly. 
* Platform or Cluster teams can check for the utilization and performance of the storage per node and decide on expansion and spreading out of the data engines 
* Infrastructure team will be responsible for planning the expansion or optimizations based on the utilization of the resources.


