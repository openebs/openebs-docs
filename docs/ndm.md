---
id: ndm
title: Node Disk Manager
sidebar_label: NDM
---
------



Node Disk Manager(NDM) is an important component in the OpenEBS architecture. NDM treats block devices as resources that need to be monitored and managed just like other resources such as CPU, Memory and Network. It is a daemonset which runs on each node, detects attached block devices based on the filters and loads them as block devices custom resource into Kubernetes. These custom resources are aimed towards helping hyper-converged Storage Operators by providing abilities like:

- Easy to access inventory of Block Devices available across the Kubernetes Cluster.
- Predict failures on the Disks to help with taking preventive actions.
- Allow dynamically attaching/detaching disks to a storage pod, without restarting the corresponding NDM pod running on the Node where the disk is attached/detached. 



In spite of doing all of the above, NDM contributes to overall ease of provisioning persistent volumes. 



<br>

<a href="/docs/assets/svg/ndm.svg" target="_blank"><img src="/docs/assets/svg/ndm.svg" alt="NDM Architecture" style="width:80%"></a>

<br>

NDM is deployed as a daemonset during installation of OpenEBS. NDM daemonset discovers the disks on each node and creates a custom resource called Block Device or BD.



## Privileged access

NDM daemon runs in containers and has to access the underlying storage devices and run in Privileged mode. NDM requires privileged mode because it requires access to /dev, /proc and /sys directories for monitoring the attached devices and also to fetch the details of the attached device using various probes. NDM is responsible for the discovery of block devices and filtering out devices that should not be used by OpenEBS; for example, detecting the disk that has OS filesystem. NDM pod by default mounts the `/proc` directory of the host inside the container and then load the `/proc/1/mounts` file to find the disk used by OS.

To allow OpenEBS to run in privileged mode in `selinux=on` nodes, the cluster should be configured to grant privileged access to OpenEBS service account.





## NDM daemonset functions:

- *Discover* block devices attached to a Kubernetes Node
  - Discover block devices on startup - create and/or update status.
  - Maintain cluster-wide unique id of the disk using the following scheme:
    - md5 hash of WWN / PartitionUUID / FileSystemUUID / DeviceMapperUUID.
- Detect block device addition/removal from a node and update the status of Block device.
- Add `blockDevice` as Kubernetes custom resource with following properties:
  - spec: The following will be updated if they are available.
    - Device Path
    - Device Links (by id, by name)
    - Vendor and Model information
    - WWN and Serial
    - Capacity
    - Sector and Block Size
  - labels:
    - hostname (kubernetes.io/hostname)
    - blockdevice-type (ndm.io/blockdevice-type)
    - Managed (ndm.io/managed)
  - status can have the following values:
    - Active : Block device is available on the node
    - Inactive : Block device is not available on the given node anymore
    - Unknown : NDM was stopped on the node where Block device was last detected / not able to determine the status

## Filters:

- Configure filters for the type of block device to be created as blockdevice CR. The filters can be configured either via vendor type or via device path patterns or mount point.
- Filters are of either `include filters` or `exclude filters` . They are configured as configmap. Admin user can configure these filters at the time of OpenEBS installation by changing the NDM configmap either in the OpenEBS operator yaml file or in the helm `values.yaml` file. If these filters need to be updated after the installation, then one of the following methods can be followed.
  - If OpenEBS is installed using operator.yaml file, update the filters in the configmap and apply the operator.yaml
  - If OpenEBS is installed using helm, update the filters in the configmap of values.yaml and do the helm upgrade
  - Or, directly edit NDM configmap using `kubectl edit` and update the filters

More details can be found from [here](/docs/next/ugndm.html).

## NDM Roadmap:

- **Auto provisioning of disks using CSI drivers of external storage:** NDM provisioner will invoke the NDM agent which inturn will initiate the provisioning of an external disk through a corresponding CSI driver



<br>

<hr>

<br>

## See Also:

### [OpenEBS Architecture](/docs/next/architecture.html)

### [Local PV User Guide](/docs/next/uglocalpv-device.html)

### [cStor User Guide](/docs/next/ugcstor.html)

### [Understanding Disk Mount Status on Node](/docs/next/faq.html#what-must-be-the-disk-mount-status-on-node-for-provisioning-openebs-volume)



