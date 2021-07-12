---
id: ugndm
title: Node Disk Manager User Guide
sidebar_label: NDM
---
------

<br>

<a href="/docs/assets/svg/2-config-sequence.svg" target="_blank"><img src="/docs/assets/svg/2-config-sequence.svg" alt="OpenEBS configuration flow" style="width:100%"></a>

<br>

This section provides the operations need to performed by the Admin for configuring NDM.

## Admin operations

[Include filters](#Include-filters)

[Exclude filters](#Exclude-filters)

[Create blockdevice CRs for unsupported disks](#create-blockdevice-CRs-for-partitioned-disks) 



<h2><a class="anchor" aria-hidden="true" id="admin-operations"></a>Admin Operations</h2>
<hr>

<h3><a class="anchor" aria-hidden="true" id="Include-filters"></a>Include filters</h3>

Users can include only selected block device for the creation of blockdevice CR and then use only the created blockdevice CR for cStor pool creation or using for provisioning Local PV based on device. For including the selected blockdevices, update the operator YAML file with the required blockdevices under NDM configuration section so that only these blockdevice will be taken for the creation of blockdevice CR. Add the blockdevice path in the following configuration for specifying particular disks. This configuration must be added in `openebs-ndm-config` under `Configmap` in `openebs-operator.yaml`. 

This change must be done in the `openebs-operator.yaml` file that you have downloaded before OpenEBS installation. If the change is performed after the OpenEBS installation, then user must restart corresponding NDM DaemonSet pods to update the NDM configuration. 


```
filterconfigs:
- key: path-filter
  name: path filter
  state: true
  include: "/dev/sda"
  exclude: ""         
```

When the above configuration is used, only the block device `/dev/sda` will be used for creating the block device custom resource. All other disks will be excluded. 

**Note**:

- Regex support is not available on the `filterconfigs` in NDM `Configmap` and the `Configmap` is applicable to cluster level. This means, if user provide `/dev/sdb` in `filterconfigs` as an include filter, then all `/dev/sdb` blockdevices from all nodes in the cluster will be used for the creation of blockdevice CR by NDM.



<h3><a class="anchor" aria-hidden="true" id="Exclude-filters"></a>Exclude filters</h3>

NDM do some filtering on the disks to exclude, for example boot disk. By default, NDM excludes the following device path while creating block device CR. This configuration is added in `openebs-ndm-config` under `Configmap` in `openebs-operator.yaml`.

```
/dev/loop - loop devices.
/dev/fd - file descriptors.
/dev/sr - CD-ROM devices.
/dev/ram - ramdisks.
/dev/dm -lvm.
/dev/md -multiple device ( software RAID devices).
/dev/rbd - ceph RBD devices
/dev/zd - zfs volumes
```

The following is the snippet of NDM configuration file from openebs operator YAML which excludes the provided disks/paths.

```
filterconfigs:
  - key: os-disk-exclude-filter
    name: os disk exclude filter
    state: true
    exclude: "/,/etc/hosts,/boot"
  - key: vendor-filter
    name: vendor filter
    state: true
    include: ""
    exclude: "CLOUDBYT,OpenEBS"
  - key: path-filter
    name: path filter
    state: true
    include: ""
    exclude: "loop,/dev/fd0,/dev/sr0,/dev/ram,/dev/dm-,/dev/md,/dev/rbd"    
```

It is also possible to customize by adding more disk types associated with your nodes. For example, used disks, unwanted disks and so on. 

This change must be done in the `openebs-operator.yaml` file that you have downloaded before OpenEBS installation. If the change is performed after the OpenEBS installation, then user must restart corresponding NDM DaemonSet pods to update the NDM configuration.

```
filterconfigs:
  - key: path-filter
    name: path filter
    state: true
    include: ""
    exclude: "loop,/dev/fd0,/dev/sr0,/dev/ram,/dev/dm-,/dev/md,/dev/sdb"
```

**Note:** 

- Regex support is not available on the  `filterconfigs` in NDM `Configmap` and the Configmap is applicable to cluster level. This means, if user provide `/dev/sdb` in configmap as an excluded filter, then all `/dev/sdb` blockdevices from all nodes in the cluster will be excluded by NDM.

- It is recommended to use OpenEBS provisioner alone in the cluster. If you are using other storage provider provisioner like `gce-pd` along with OpenEBS, use exclude filters to avoid those disks from being consumed by OpenEBS. For example, if you are using the `standard` storage class in GKE with storage provisioner as **kubernetes.io/gce-pd**, and when it creates a PVC, a GPD is attached to the node. This GPD will be detected by NDM and it may be used by OpenEBS for provisioning volume. To avoid this scenario, it is recommended to put the associated device path created on the node in the **exclude** field under **path-filter**. If GPD is attached as `/dev/sdc` , then add `/dev/sdc` in the above mentioned field.

  **Example snippet:**

  In the downloaded openebs-operator.yaml, find *openebs-ndm-config* configmap and update the values for **path-filter** and any other  filters if required.

  ```
  ---
  # This is the node-disk-manager related config.
  # It can be used to customize the disks probes and filters
  apiVersion: v1
  kind: ConfigMap
  metadata:
   name: openebs-ndm-config
   namespace: openebs
  data:
   # udev-probe is default or primary probe which should be enabled to run ndm
   # filterconfigs contains configs of filters - in their form fo include
   # and exclude comma separated strings
   node-disk-manager.config: |
     probeconfigs:
       - key: udev-probe
         name: udev probe
         state: true
       - key: seachest-probe
         name: seachest probe
         state: false
       - key: smart-probe
         name: smart probe
         state: true
     filterconfigs:
       - key: os-disk-exclude-filter
         name: os disk exclude filter
         state: true
         exclude: "/,/etc/hosts,/boot"
       - key: vendor-filter
         name: vendor filter
         state: true
         include: ""
         exclude: "CLOUDBYT,OpenEBS"
       - key: path-filter
         name: path filter
         state: true
         include: ""
         exclude: "loop,/dev/fd0,/dev/sr0,/dev/ram,/dev/dm-,/dev/md,/dev/sdc"
  ---
  ```



<h3><a class="anchor" aria-hidden="true" id="create-blockdevice-CRs-for-partitioned-disks"></a>Create blockdevice CRs for unsupported disks</h3>

Currently, NDM out of the box manages disks, partitions, lvm, crypt and other dm devices. If the user need to have blockdevice for other device types like md array or 
any other unsupported device types, the blockdevice resource can be manually created using the following steps:

1. Create the sample block device CR YAML using the following spec. Following is the sample block device CR YAML.

   ```
    apiVersion: openebs.io/v1alpha1
    kind: BlockDevice
    metadata:
      name: example-blockdevice-1
      labels:
        kubernetes.io/hostname: <host name in which disk/blockdevice is attached> # like gke-openebs-user-default-pool-044afcb8-bmc0
        ndm.io/managed: "false" # for manual blockdevice creation put false
        ndm.io/blockdevice-type: blockdevice
    status:
      claimState: Unclaimed
      state: Active
    spec:
      capacity:
        logicalSectorSize: 512
        storage: <total capacity in bytes> #like 53687091200
      details:
        deviceType: <device type> # like disk, partition, lvm, crypt, md
        firmwareRevision: <firmware revision>
        model: <model name of blockdevice> # like PersistentDisk
        serial: <serial no of disk> # like google-disk-2
        compliance: <compliance of disk> #like "SPC-4"
        vendor: <vendor of disk> #like Google
      devlinks:
      - kind: by-id
        links:
        - <link1> # like /dev/disk/by-id/scsi-0Google_PersistentDisk_disk-2
        - <link2> # like /dev/disk/by-id/google-disk-2
      - kind: by-path
        links:
        - <link1> # like /dev/disk/by-path/virtio-pci-0000:00:03.0-scsi-0:0:2:0
      nodeAttributes:
        nodeName: <node name> # output of `kubectl get nodes` can be used
      path: <devpath> # like /dev/md0

   ```

2. Modify the created block device CR sample YAML with the disk information. In the above block device CR sample spec, following fields must be filled before applying the YAML.

   - name
     - Provide unique name for the blockdevice CR. In the above YAML spec, given name for the blockdevice CR is `example-blockdevice-1`
   - kubernetes.io/hostname
     - Hostname of the node where the blockdevice is attached. 
   - storage
      - Provide the storage capacity in `bytes` like `53687091200`.
   - logicalSectorSize
     - logical sector size of blockdevice. For example, 512, 4096 etc. Provided 512 in the above example snippet. This value can be changed as per the logical sector size of the device.
   - deviceType
     - Type of the device. This can be obtained from `lsblk` output. eg: lvm, crypt, nbd, md etc
   - links
     - This field should be filled for by-id and by-path. These details can be obtained from worker node by running the following command `udevadm info -q property -n <device_path>` 
   - nodeName
     - Name of the Node where the blockdevice is attached.  The output of `kubectl get nodes` can be used to obtain this value.
   - path
     - The value should be like `/dev/dm-0` or `/dev/md0`.
   
3. Apply the modified YAML file to create the blockdevice CR for the provided device path. 
   
   ```
   kubectl apply -f <blockdevice-cr.yaml> -n <openebs_namespace>
   ```

   **Note:** The blockdevice CR should be created on the same namespace where openebs is installed.

4. Repeat the same steps for each unsupported device and create blockdevice CR for the devices.

5. Verify if the blockdevice is created by running the following `kubectl get blockdevice -n openebs` command.

**Note:** If you are creating a block device CR for an unsupported device, then you must add the corresponding disk under **exclude** filter so that NDM will not select the particular disk for BD creation. See [here](#Exclude-filters) for customizing the exclude filter in NDM configuration.

<br>

## See Also:


### [Understanding Node Disk Manager](/docs/next/ndm.html)


