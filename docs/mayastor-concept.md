---
id: mayastor
title: Mayastor
sidebar_label: Mayastor
---
----

## What is Mayastor?

**Mayastor** is currently under development as a sub-project of the Open Source CNCF project [**OpenEBS**](https://openebs.io/).  OpenEBS is a "Container Attached Storage" or CAS solution which extends Kubernetes with a declarative data plane, providing flexible, persistent storage for stateful applications.

Design goals for Mayastor include:

* Highly available, durable persistence of data.
* To be readily deployable and easily managed by autonomous SRE or development teams.
* To be a low-overhead abstraction.

OpenEBS Mayastor incorporates Intel's [Storage Performance Development Kit](https://spdk.io/).  It has been designed from the ground up to leverage the protocol and compute efficiency of NVMe-oF semantics,  and the performance capabilities of the latest generation of solid-state storage devices, in order to deliver a storage abstraction with performance overhead measured to be within the range of single-digit percentages. 

By comparison, most pre-CAS shared everything storage systems are widely thought to impart an overhead of at least 40% and sometimes as much as 80% or more than the capabilities of the underlying devices or cloud volumes. Additionally, pre-CAS shared storage scales in an unpredictable manner as I/O from many workloads interact and complete for the capabilities of the shared storage system.  

While Mayastor utilizes NVMe-oF, it does not require NVMe devices or cloud volumes to operate, as is explained below.  



>**Mayastor is beta software**.  It is considered largely, if not entirely, feature complete and substantially without major known defects.  Minor and unknown defects can be expected; **please deploy accordingly**.

### Basic Architecture

The objective of this section is to provide the user and evaluator of Mayastor with a topological view of the gross anatomy of a Mayastor deployment.   A description will be made of the expected pod inventory of a correctly deployed cluster, the roles and functions of the constituent pods and related Kubernetes resource types, and the high-level interactions between them and the orchestration thereof.

More detailed guides to Mayastor's components, their design and internal structure, and instructions for building Mayastor from source, are maintained within the [project's GitHub repository](https://github.com/openebs/Mayastor).

### Topology
<a href="../assets/mayastor_basic_cluster_topology.png" target="_blank"><img src="../assets/mayastor_basic_cluster_topology.png"></a>


### Dramatis Personae

| Name | Resource Type | Function | Occurrence in Cluster |
| :--- | :--- | :--- | :--- |
| **moac** | Pod | Hosts control plane containers | Single |
| moac | Service | Exposes MOAC REST service end point | Single |
| moac | Deployment | Declares desired state for the MOAC pod | Single |
|   |   |   |   |
| **mayastor-csi** | Pod | Hosts CSI Driver node plugin containers | All worker nodes |
| mayastor-csi | DaemonSet | Declares desired state for mayastor-csi pods | Single |
|   |    |   |   |
| **mayastor** | Pod | Hosts Mayastor I/O engine container | User-selected nodes |
| mayastor | DaemonSet | Declares desired state for Mayastor pods | Single |
|   |   |   |   |
| **nats** | Pod | Hosts NATS Server container | Single |
| nats | Deployment | Declares desired state for NATS pod | Single |
| nats | Service | Exposes NATS message bus end point | Single |
|   |   |   |   |
| **mayastornodes** | CRD | Inventories and reflects the state of Mayastor pods | One per Mayastor Pod |
| **mayastorpools** | CRD | Declares a Mayastor pool's desired state and reflects its current state | User-defined, zero to many |
| **mayastorvolumes** | CRD | Inventories and reflects the state of Mayastor-provisioned Persistent Volumes | User-defined, zero to many |

### Component Roles

#### MOAC

A Mayastor deployment features a single MOAC pod, declared via a Deployment resource of the same name and has its API's gRPC endpoint exposed via a cluster Service, also of the same name.  The MOAC pod is the principle control plane actor and encapsulates containers for both the Mayastor CSI Driver's controller implementation \(and its external-attacher sidecar\) and the MOAC component itself.

The MOAC component implements the bulk of the Mayastor-specific control plane.  It is called by the CSI Driver controller in response to dynamic volume provisioning events, to orchestrate the creation of a nexus at the Mayastor pod of an appropriate node and also the creation of any additional data replicas on other nodes as may be required to satisfy the desired configuration state of the PVC  \(i.e. replication factor &gt; 1\).  MOAC is also responsible for creating and reporting Storage Pools, for which purpose it implements a watch on the Kubernetes API server for relevant custom resource objects \(mayastorpools.openebs.io\).

MOAC exposes a REST API endpoint on the cluster using a Kubernetes Service of the same name.  This is currently used to support the export of volume metrics to Prometheus/Grafana, although this mechanism will change in later releases.

#### Mayastor

The Mayastor pods of a deployment are its principle data plane actors, encapsulating the Mayastor containers which implement the I/O path from the block devices at the persistence layer, up to the relevant initiators on the worker nodes mounting volume claims.

The instance of the `mayastor` binary running inside the container performs four major classes of functions:

* Present a gRPC interface to the MOAC control plane component, to allow the latter to orchestrate creation, configuration and deletion of Mayastor managed objects hosted by that instance
* Create and manage storage pools hosted on that node
* Create, export and manage nexus objects \(and by extension, volumes\) hosted on that node
* Create and share "replicas" from storage pools hosted on that node
  * Local replica -&gt; loopback -&gt; Local Nexus
  * Local replica - &gt; NVMe-F TCP -&gt;  Remote Nexus \(hosted by a Mayastor container on another node\)
  * Remote replicas are employed by a Nexus as synchronous data copies, where replication is in use

When a Mayastor pod starts running, an init container attempts to verify connectivity to the NATS message bus in the Mayastor namespace.  If a connection can be established the Mayastor container is started, and the Mayastor instance performs registration with MOAC over the message bus.  In this way, MOAC maintains a registry of nodes \(specifically, running Mayastor instances\) and their current state.  For each registered Mayastor container/instance, MOAC creates a MayastorNode custom resource within the Kubernetes API of the cluster.

The scheduling of Mayastor pods is determined declaratively by using a DaemonSet specification.  By default, a `nodeSelector` field is used within the pod spec to select all worker nodes to which the user has attached the label `openebs.io/engine=mayastor` as recipients of a Mayastor pod.   It is in this way that the MayastorNode count and location is set appropriate to the hardware configuration of the worker nodes \(i.e. which nodes host the block storage devices to be used\), and capacity and performance demands of the cluster.

#### Mayastor-CSI

The mayastor-csi pods within a cluster implement the node plugin component of Mayastor's CSI driver.  As such, their function is to orchestrate the mounting of Mayastor provisioned volumes on worker nodes on which application pods consuming those volumes are scheduled.  By default a mayastor-csi pod is scheduled on every node in the target cluster, as determined by a DaemonSet resource of the same name.  These pods each encapsulate two containers, `mayastor-csi` and `csi-driver-registrar`

 It is not necessary for the node plugin to run on every worker node within a cluster and this behaviour can be modified if so desired through the application of appropriate node labeling and the addition of a corresponding  `nodeSelector` entry within the pod spec of the mayastor-csi DaemonSet.  It should be noted that if a node does not host a plugin pod, then it will not be possible to schedule pod on it which is configured to mount Mayastor volumes.

Further detail regarding the implementation of CSI driver components and their function can be found within the Kubernetes CSI Developer Documentation.

#### NATS

NATS is a high performance open source messaging system.  It is used within Mayastor as the transport mechanism for registration messages passed between Mayastor I/O engine pods running in the cluster and the MOAC component which maintains an inventory of active Mayastor nodes and reflects this via CRUD actions on MayastorNode custom resources.

In future releases of Mayastor, the control plane will transition towards a more microservice-like architecture following the saga pattern, whereupon a highly available NATS deployment within the Mayastor namespace will be employed as an event bus.
<br><br>
----
## Scope

This quickstart guide describes the actions necessary to perform a basic installation of Mayastor on an existing Kubernetes cluster, sufficient for evaluation purposes.  It assumes that the target cluster will pull the Mayastor container images directly from MayaData public container repositories.  Where preferred, it is also possible to [build Mayastor locally from source](https://github.com/openebs/Mayastor/blob/develop/doc/build.md) and deploy the resultant images but this is outside of the scope of this quickstart document.

Deploying and operating Mayastor within production contexts requires a foundational knowledge of Mayastor internals and best practices, found elsewhere within this documentation resource.  Some application use cases may require specific configuration and where this is so, it is called out in the Use Cases section.


>**Mayastor is beta software**.  It is considered largely, if not entirely, feature complete.  Beta software "will generally have many more bugs in it than completed software and speed or performance issues, and may still cause crashes or data loss"
----

## Known Issues

### Live Issue Tracker

Mayastor is currently considered to be beta software.

> "(it) will generally have many more bugs in it than completed software and speed or performance issues, and may still cause crashes or data loss."

The project's maintainers operate a live issue tracking dashboard for defects which they have under active triage and investigation.  It can be accessed [here](https://mayadata.atlassian.net/secure/Dashboard.jspa?selectPageId=10015).  You are strongly encouraged to familiarize yourself with the issues identified there before using Mayastor and when raising issue reports in order to limit to the extent possible redundant issue reporting. 

### How is Mayastor Tested?

Mayastor's maintainers perform integration and end-to-end testing on nightly builds and named releases.  Clusters used to perform this testing are composed of worker nodes running Ubuntu 20.04.2 LTS, using the docker runtime 20.10.5 under Kubernetes version 1.19.8.  Other testing efforts are underway including soak testing and failure injection testing.

We periodically access the labs of partners and community members for scale and performance testing and would welcome offers of any similar or other testing assistance.  

### Common Installation Issues

#### A Mayastor pod restarts unexpectedly with exit code 132 whilst mounting a PVC

The Mayastor process has been sent the SIGILL signal as the result of attempting to execute an illegal instruction.  This indicates that the host node's CPU does not satisfy the prerequisite instruction set level for Mayastor \(SSE4.2 on x86-64\).


#### Deploying Mayastor on RKE & Fedora CoreOS

In addition to ensuring that the general prerequisites for installation are met, it is necessary to add the following directory mapping to the `services_kubelet->extra_binds` section of the cluster's`cluster.yml file.`

```text
/opt/rke/var/lib/kubelet/plugins:/var/lib/kubelet/plugins
```

If this is not done, CSI socket paths won't match expected values and the Mayastor CSI driver registration process will fail, resulting in the inability to provision Mayastor volumes on the cluster.

### Other Issues

#### Lengthy worker node reboot times

When rebooting a node that runs applications mounting Mayastor volumes, this can take tens of minutes. The reason is the long default NVMe controller timeout (`ctrl_loss_tmo`). The solution is to follow the best k8s practices and cordon the node ensuring there aren't any application pods running on it before the reboot.
<br><br>
----

## Known Limitations


### Volume and Pool Capacity Expansion

Once provisioned, neither Mayastor Disk Pools nor Mayastor Volumes can be re-sized.  A Mayastor Pool can have only a single block device as a member.  Mayastor Volumes are exclusively thick-provisioned.

### Snapshots and Clones

Mayastor has no snapshot or cloning capabilities.

### Volumes are "Highly Durable" but without multipathing are not "Highly Available"

Mayastor Volumes can be configured (or subsequently re-configured) to be composed of 2 or more "children" or "replicas"; causing synchronously mirrored copies of the volume's data to be maintained on more than one worker node and Disk Pool.  This contributes additional "durability" at the persistence layer, ensuring that viable copies of a volume's data remain even if a Disk Pool device is lost.

However a Mayastor volume is currently accessible to an application only via a single target instance (NVMe-oF, or iSCSI) of a single Mayastor pod.  If that pod terminates (through the loss of the worker node on which it's scheduled, execution failure, pod eviction etc.) then there will be no viable I/O path to any remaining healthy replicas and access to data on the volume cannot be maintained.

There has been initial discovery work completed in supporting and testing the use of multipath connectivity to Mayastor pods.  The work of developing and supporting production usage of multipath connectivity is currently scheduled to complete after general availability.
