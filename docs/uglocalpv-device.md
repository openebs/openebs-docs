---
id: uglocalpv-device
title: OpenEBS Local PV Device User Guide
sidebar_label: Local PV Device
---
------

<br>

<a href="/docs/assets/svg/4-config-sequence.svg" target="_blank"><img src="/docs/assets/svg/4-config-sequence.svg" alt="OpenEBS configuration flow" style="width:100%"></a>

<br>

This guide will help you to set up and use OpenEBS Local Persistent Volumes backed by Block Devices.

*OpenEBS Dynamic Local PV provisioner* can create Kubernetes Local Persistent Volumes using block devices available on the node to persist data, hereafter referred to as *OpenEBS Local PV Device* volumes. 

*OpenEBS Local PV Device* volumes have the following advantages compared to native Kubernetes Local Persistent Volumes. 
- Dynamic Volume provisioner as opposed to a Static Provisioner. 
- Better management of the Block Devices used for creating Local PVs by OpenEBS NDM. NDM provides capabilities like discovering Block Device properties, setting up Device Filters, metrics collection and ability to detect if the Block Devices have moved across nodes. 

OpenEBS Local PV uses volume topology aware pod scheduling enhancements introduced by [Kubernetes Local Volumes](https://kubernetes.io/docs/concepts/storage/volumes/#local)

<br>

:::tip QUICKSTART

OpenEBS Local PV Device volumes will be created using the Block Devices available on the node. You can customize which block devices can be used for creating Local PVs by [configuring NDM parameters](#install) and/or by creating new [StorageClass](#create-storageclass). 

If you have OpenEBS already installed, you can create an example pod that persists data to *OpenEBS Local PV Device* with following kubectl commands. 
```
kubectl apply -f https://openebs.github.io/charts/examples/local-device/local-device-pvc.yaml
kubectl apply -f https://openebs.github.io/charts/examples/local-device/local-device-pod.yaml
```

Verify using below kubectl commands that example pod is running and is using a OpenEBS Local PV Device.
```
kubectl get pod hello-local-device-pod
kubectl get pvc local-device-pvc
```

For a more detailed walkthrough of the setup, follow along the rest of this document.
:::

## Minimum Versions

- Kubernetes 1.12 or higher is required
- OpenEBS 1.0 or higher is required.

:::note air-gapped environment
If you are running your Kubernetes cluster in an air-gapped environment, make sure the following container images are available in your local repository.
- openebs/localpv-provisioner
- openebs/linux-utils
- openebs/node-disk-manager
- openebs/node-disk-operator
:::

## Prerequisites

For provisioning Local PV using the block devices, the Kubernetes nodes should have block devices attached to the nodes. The block devices can optionally be formatted and mounted. 

The block devices can be any of the following:

- SSD, NVMe or Hard Disk attached to a Kubernetes node (Bare metal server)
- Cloud Provider Disks like EBS or GPD attached to a Kubernetes node (Cloud instances. GKE or EKS) 
- Virtual Disks like a vSAN volume or VMDK disk attached to a Kubernetes node (Virtual Machine)

## Install

### Customize NDM and Install

You can skip this section if you have already installed OpenEBS.

*OpenEBS Dynamic Local Provisioner* uses the Block Devices discovered by NDM to create Local PVs. NDM offers some configurable parameters that can be applied during the OpenEBS Installation. Some key configurable parameters available for NDM are:

1. Prepare to install OpenEBS by providing custom values for configurable parameters.
   - The location of the *OpenEBS Dynamic Local PV provisioner* container image.
     <div class="co">
     Default value: openebs/provisioner-localpv
     YAML specification: spec.image on Deployment(localpv-provisioner)
     Helm key: localprovisioner.image
     </div>  

   - The location of the *OpenEBS NDM DaemonSet* container image. NDM DaemonSet helps with discovering block devices attached to a node and creating Block Device Resources.
     <div class="co">
     Default value: openebs/node-disk-manager
     YAML specification: spec.image on DaemonSet(openebs-ndm)
     Helm key: ndm.image
     </div>  

   - The location of the *OpenEBS NDM Operator* container image. NDM Operator helps with allocating Block Devices to Block Device Claims raised by *OpenEBS Dynamic Local PV Provisioner*. 
     <div class="co">
     Default value: openebs/node-disk-operator
     YAML specification: spec.image on Deployment(openebs-ndm-operator)
     Helm key: ndmOperator.image
     </div>  

   - The location of the *Provisioner Helper* container image. *OpenEBS Dynamic Local Provisioner* create a *Provisioner Helper* pod to clean up the data from the block device after the PV has been deleted.
     <div class="co">
     Default value: openebs/linux-utils
     YAML specification: Environment Variable (CLEANUP_JOB_IMAGE) on Deployment(ndm-operator) 
     Helm key: helper.image
     </div>

   - Specify the list of block devices for which BlockDevice CRs must be created. A comma separated values of path regular expressions can be specified. 
     <div class="co">
     Default value: all
     YAML specification: data."node-disk-manager.config".filterconfigs.key["path-filter"].include on ConfigMap(openebs-ndm-config)
     Helm key: ndm.filters.includePaths
     </div>

   - Specify the list of block devices for which BlockDevice CRs must not be created. A comma separated values of path regular expressions can be specified. 
     <div class="co">
     Default value: "loop,fd0,sr0,/dev/ram,/dev/dm-,/dev/md"
     YAML specification: data."node-disk-manager.config".filterconfigs.key["path-filter"].exclude on ConfigMap(openebs-ndm-config)
     Helm key: ndm.filters.excludePaths
     </div>

2. You can proceed to install OpenEBS either using kubectl or helm using the steps below. 

   - Install using kubectl
  
     If you would like to change the default values for any of the configurable parameters mentioned in the previous step, download the `openebs-operator.yaml` and make the necessary changes before applying. 
     ```
     kubectl apply -f https://openebs.github.io/charts/openebs-operator.yaml
     ```

     :::note 
     If you would like to use only Local PV (hostpath and device), you can install a lite version of OpenEBS using the following command.

     kubectl apply -f https://openebs.github.io/charts/openebs-operator-lite.yaml
     kubectl apply -f https://openebs.github.io/charts/openebs-lite-sc.yaml
     :::

   - Install using OpenEBS helm charts
  
     If you would like to change the default values for any of the configurable parameters mentioned in the previous step, specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.
  
     ```
     helm repo add openebs https://openebs.github.io/charts
     helm repo update
     helm install --namespace openebs --name openebs openebs/openebs
     ```

### (Optional) Block Device Tagging

You can reserve block devices in the cluster that you would like the *OpenEBS Dynamic Local Provisioner* to pick up some specific block devices available on the node. You can use the NDM Block Device tagging feature to reserve the devices. For example, if you would like Local SSDs on your cluster for running Mongo stateful application. You can tag a few devices in the cluster with a tag named `mongo`.

```
kubectl label bd -n openebs blockdevice-0052b132e6c5800139d1a7dfded8b7d7 openebs.io/block-device-tag=mongo
```

## Create StorageClass

You can skip this section if you would like to use default OpenEBS Local PV Device StorageClass created by OpenEBS. 

The default Storage Class is called `openebs-device`. If the block devices are not formatted, the devices will be formatted with `ext4`. 

1. To create your own StorageClass to customize how Local PV with devices are created. For instance, if you would like to run MongoDB stateful applications with Local PV device, you would want to set the default filesystem as `xfs` and/or also dedicate some devices on node that you want to use for Local PV. Save the following StorageClass definition as `local-device-sc.yaml`

   ```
   apiVersion: storage.k8s.io/v1
   kind: StorageClass
   metadata:
     name: local-device
     annotations:
       openebs.io/cas-type: local
       cas.openebs.io/config: |
         - name: StorageType
           value: device
         - name: FSType
           value: xfs
         - name: BlockDeviceTag
           value: "mongo"
   provisioner: openebs.io/local
   reclaimPolicy: Delete
   volumeBindingMode: WaitForFirstConsumer
   ```
   :::note 
   The `volumeBindingMode` MUST ALWAYS be set to `WaitForFirstConsumer`. `volumeBindingMode: WaitForFirstConsumer` instructs Kubernetes to initiate the creation of PV only after Pod using PVC is scheduled to the node.
   :::

   :::note 
   The `FSType` will take effect only if the underlying block device is not formatted. For instance if the block device is formatted with "Ext4", specifying "XFS" in the storage class will not clear Ext4 and format with XFS. If the block devices are already formatted, you can clear the filesystem information using `wipefs -f -a <device-path>`. After the filesystem has been cleared, NDM pod on the node needs to be restarted to update the Block Device.
   :::

2. Edit `local-device-sc.yaml` and update with your desired values for:

   - `metadata.name`
   - `cas.openebs.io/config.FSType`
   - `cas.openebs.io/config.BlockDeviceTag`

   :::note 
   Block Device Tag support for Local Volumes was introduced in OpenEBS 1.9. 
   
   When specifying the value for BlockDeviceTag, you must already have Block Devices on the nodes labelled with the tag. See [Block Device Tagging](#optional-block-device-tagging)
   :::

3. Create OpenEBS Local PV Device Storage Class. 
   ```
   kubectl apply -f local-device-sc.yaml
   ```

4. Verify that the StorageClass is successfully created. 
   ```
   kubectl get sc local-device -o yaml
   ```
   
## Create a PersistentVolumeClaim

The next step is to create a PersistentVolumeClaim. Pods will use PersistentVolumeClaims to request Device backed Local PV from *OpenEBS Dynamic Local PV provisioner*.

1. Here is the configuration file for the PersistentVolumeClaim. Save the following PersistentVolumeClaim definition as `local-device-pvc.yaml`

   ```
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: local-device-pvc
   spec:
     storageClassName: local-device
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 5G
   ```

2. Create the PersistentVolumeClaim

   ```
   kubectl apply -f local-device-pvc.yaml
   ```

3. Look at the PersistentVolumeClaim:
   
   ```
   kubectl get pvc local-device-pvc
   ```

   The output shows that the `STATUS` is `Pending`. This means PVC has not yet been used by an application pod. The next step is to create a Pod that uses your PersistentVolumeClaim as a volume.

   <div class="co">
   NAME               STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS     AGE
   local-device-pvc   Pending                                      local-device   31s
   </div>

### Using Raw Block Volume

By default, Local PV volume will be provisioned with volumeMode as filesystem. If you would like to use it as [Raw Block Volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#raw-block-volume-support), specify `spec.volumeMode` as `Block` in the Persistent Volume Claim spec. Here is the configuration file for the PersistentVolumeClaim with Raw Block Volume Support. 

   ```
   kind: PersistentVolumeClaim
   apiVersion: v1
   metadata:
     name: local-device-pvc-block
   spec:
     storageClassName: local-device
     volumeMode: Block
     accessModes:
       - ReadWriteOnce
     resources:
       requests:
         storage: 5G
   ```

   :::note 
   Raw Block Volume support was introduced for OpenEBS Local PV OpenEBS 1.5. 
   :::


## Create Pod to consume OpenEBS Local PV backed by Block Device

1. Here is the configuration file for the Pod that uses Local PV. Save the following Pod definition to `local-device-pod.yaml`.

   ```
   apiVersion: v1
   kind: Pod
   metadata:
     name: hello-local-device-pod
   spec:
     volumes:
     - name: local-storage
       persistentVolumeClaim:
         claimName: local-device-pvc
     containers:
     - name: hello-container
       image: busybox
       command:
          - sh
          - -c
          - 'while true; do echo "`date` [`hostname`] Hello from OpenEBS Local PV." >> /mnt/store/greet.txt; sleep $(($RANDOM % 5 + 300)); done'
       volumeMounts:
       - mountPath: /mnt/store
         name: local-storage
   ```

   :::note 
   As the Local PV storage classes use `waitForFirstConsumer`, do not use `nodeName` in the Pod spec to specify node affinity. If `nodeName` is used in the Pod spec, then PVC will remain in `pending` state. For more details refer https://github.com/openebs/openebs/issues/2915.
   :::

2. Create the Pod:

   ```
   kubectl apply -f local-device-pod.yaml
   ```

3. Verify that the container in the Pod is running;

   ```
   kubectl get pod hello-local-device-pod
   ```

4. Verify that the container is using the Local PV Device 
   ```
   kubectl describe pod hello-local-device-pod
   ```

   The output shows that the Pod is running on `Node: gke-user-helm-default-pool-3a63aff5-1tmf` and using the persistent volume provided by `local-describe-pvc`.

   <div class="co">
   Name:         hello-local-device-pod
   Namespace:    default
   Priority:     0
   Node:         gke-user-helm-default-pool-92abeacf-89nd/10.128.0.16
   Start Time:   Thu, 16 Apr 2020 17:56:04 +0000  
   ...  
   Volumes:
     local-storage:
       Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
       ClaimName:  local-device-pvc
       ReadOnly:   false
   ...
   </div>

5. Look at the PersistentVolumeClaim again to see the details about the dynamically provisioned Local PersistentVolume
   ```
   kubectl get pvc local-device-pvc
   ```

   The output shows that the `STATUS` is `Bound`. A new Persistent Volume `pvc-79d25095-eb1f-4028-9843-7824cb82f07f` has been created. 

   <div class="co">
   NAME               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
   local-device-pvc   Bound    pvc-79d25095-eb1f-4028-9843-7824cb82f07f   5G         RWO            local-device   5m56s
   </div>

6. Look at the PersistentVolume details to see where the data is stored. Replace the PVC name with the one that was displayed in the previous step. 
   ```
   kubectl get pv pvc-79d25095-eb1f-4028-9843-7824cb82f07f -o yaml
   ```
   The output shows that the PV was provisioned in response to PVC request  `spec.claimRef.name: local-device-pvc`. 

   <div class="co">
   apiVersion: v1
   kind: PersistentVolume
   metadata:
     name: pvc-79d25095-eb1f-4028-9843-7824cb82f07f
     annotations:
       pv.kubernetes.io/provisioned-by: openebs.io/local
     ...
   spec:
     accessModes:
       - ReadWriteOnce
     capacity:
       storage: 5G
     claimRef:
       apiVersion: v1
       kind: PersistentVolumeClaim
       name: local-device-pvc
       namespace: default
       resourceVersion: "291148"
       uid: 79d25095-eb1f-4028-9843-7824cb82f07f 
     ...
     ...
     local:
       fsType: ""
       path: /mnt/disks/ssd0
     nodeAffinity:
       required:
         nodeSelectorTerms:
         - matchExpressions:
           - key: kubernetes.io/hostname
             operator: In
             values:
             - gke-user-helm-default-pool-92abeacf-89nd
     persistentVolumeReclaimPolicy: Delete
     storageClassName: local-device
     volumeMode: Filesystem
   status:
     phase: Bound
   </div>
   <br/>

:::note 
A few important characteristics of a *OpenEBS Local PV* can be seen from the above output: 
- `spec.nodeAffinity` specifies the Kubernetes node where the Pod using the local volume is scheduled. 
- `spec.local.path` specifies the path of the block device associated with this PV.
:::

7. *OpenEBS Dynamic Local Provisioner* would have created a BlockDeviceClaim to get a BlockDevice from NDM. The BlockDeviceClaim will be having the same name as the PV name. Look at the BlockDeviceClaim details to see which Block Device is being used. Replace the PVC Name in the below command with the PVC name that was displayed in the previous step. 
   ```
   kubectl get bdc -n openebs bdc-pvc-79d25095-eb1f-4028-9843-7824cb82f07f
   ```

   The output shows that the `PHASE` is `Bound`, and provides the name of the Block Device `blockdevice-d1ef1e1b9dccf224e000c6f2e908c5f2`

   <div class="co">
   NAME                                           BLOCKDEVICENAME                                PHASE   AGE
   bdc-pvc-79d25095-eb1f-4028-9843-7824cb82f07f   blockdevice-d1ef1e1b9dccf224e000c6f2e908c5f2   Bound   12m
   </div>

8. Look at the BlockDevice details to see where the data is stored. Replace the BDC name with the one that was displayed in the previous step. 
   ```
   kubectl get bd -n openebs blockdevice-d1ef1e1b9dccf224e000c6f2e908c5f2 -o yaml
   ```
   The output shows that the BD is on the node `spec.nodeAttributes.nodeName: gke-user-helm-default-pool-92abeacf-89nd`. 

   <div class="co">
   apiVersion: openebs.io/v1alpha1
   kind: BlockDevice
   metadata:
     name: blockdevice-d1ef1e1b9dccf224e000c6f2e908c5f2
     namespace: openebs
   ...
   spec:
     capacity:
       logicalSectorSize: 4096
       physicalSectorSize: 4096
       storage: 402653184000
     claimRef:
       apiVersion: openebs.io/v1alpha1
       kind: BlockDeviceClaim
       name: bdc-pvc-79d25095-eb1f-4028-9843-7824cb82f07f
       namespace: openebs
       uid: 8efe7480-9117-4f51-b271-84ee51a94684
     details:
       compliance: SPC-4
       deviceType: disk
       driveType: SSD
       hardwareSectorSize: 4096
       logicalBlockSize: 4096
       model: EphemeralDisk
       physicalBlockSize: 4096
       serial: local-ssd-0
       vendor: Google
     devlinks:
     - kind: by-id
       links:
       - /dev/disk/by-id/scsi-0Google_EphemeralDisk_local-ssd-0
       - /dev/disk/by-id/google-local-ssd-0
     - kind: by-path
       links:
       - /dev/disk/by-path/pci-0000:00:04.0-scsi-0:0:1:0
     filesystem:
       fsType: ext4
       mountPoint: /mnt/disks/ssd0
     nodeAttributes:
       nodeName: gke-user-helm-default-pool-92abeacf-89nd
     partitioned: "No"
     path: /dev/sdb
     status:
       claimState: Claimed
       state: Active 
   </div>
   <br/>

:::note 
A few important details from the above Block Device are:
- `spec.filesystem` indicates if the BlockDevice has been formatted and the path where it has been mounted. 
  - If the block device is pre-formatted as in the above case, the PV will be created with path as `spec.filesystem.mountPoint`.
  - If the block device is not formatted, it will be formatted with the filesystem specified in the PVC and StorageClass. Default is `ext4`.
:::


## Cleanup

Delete the Pod, the PersistentVolumeClaim and StorageClass that you might have created. 

```
kubectl delete pod hello-local-device-pod
kubectl delete pvc local-device-pvc
kubectl delete sc local-device
```

Verify that the PV that was dynamically created is also deleted. 
```
kubectl get pv
```

## Backup and Restore

OpenEBS Local Volumes can be backed up and restored along with the application using [Velero](https://velero.io).

:::note
The following steps assume that you already have Velero with Restic integration is configured. If not, please follow the [Velero Documentation](https://velero.io/docs/) to proceed with install and setup of Velero.  If you encounter any issues or have questions, talk to us on the [#openebs channel on the Kubernetes Slack server](https://kubernetes.slack.com/messages/openebs/).
:::

### Backup

The following steps will help you to prepare and backup the data from the volume created for the example pod (`hello-local-device-pod`), with the volume mount (`local-storage`). 

1. Prepare the application pod for backup. Velero uses Kubernetes labels to select the pods that need to be backed up. Velero uses annotation on the pods to determine which volumes need to be backed up. For the example pod launched in this guide, you can inform velero to backup by specifying the following label and annotation. 
   
   ```
   kubectl label pod hello-local-device-pod app=test-velero-backup
   kubectl annotate pod hello-local-device-pod backup.velero.io/backup-volumes=local-storage
   ```
2. Create a Backup using velero. 

   ```
   velero backup create bbb-01 -l app=test-velero-backup
   ```

3. Verify that backup is successful. 

   ```
   velero backup describe bbb-01 --details
   ```

   On successful completion of the backup, the output of the backup describe command will show the following:
   <div class="co">
   ...
   Restic Backups:
     Completed:
       default/hello-local-device-pod: local-storage
   </div>
 
### Restore

1. Install and Setup Velero, with the same provider where backups were saved. Verify that backups are accessible. 
   
   ```
   velero backup get
   ```
   
   The output of should display the backups that were taken successfully. 
   <div class="co">
   NAME     STATUS      CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
   bbb-01   Completed   2020-04-25 15:49:46 +0000 UTC   29d       default            app=test-velero-backup
   </div>

2. Restore the application. 

   :::note
   Local PVs are created with node affinity. As the node names will change when a new cluster is created, create the required PVC(s) prior to proceeding with restore.
   :::
   
   Replace the path to the PVC yaml in the below commands, with the PVC that you have created. 
   ```
   kubectl apply -f https://openebs.github.io/charts/examples/local-device/local-device-pvc.yaml
   velero restore create rbb-01 --from-backup bbb-01 -l app=test-velero-backup
   ```

3. Verify that application is restored.

   ```
   velero restore describe rbb-01
   ```
   
   Depending on the data, it may take a while to initialize the volume. On successful restore, the output of the above command should show:
   <div class="co">
   ...
   Restic Restores (specify --details for more information):
     Completed:  1
   </div>
   
4. Verify that data has been restored. The application pod used in this example, write periodic messages (greetings) to the volume. 

   ```
   kubectl exec hello-local-device-pod -- cat /mnt/store/greet.txt
   ```
   
   The output will show that backed up data as well as new greetings that started appearing after application pod was restored. 
   <div class="co">
   Sat Apr 25 15:41:30 UTC 2020 [hello-local-device-pod] Hello from OpenEBS Local PV.
   Sat Apr 25 15:46:30 UTC 2020 [hello-local-device-pod] Hello from OpenEBS Local PV.
   Sat Apr 25 16:11:25 UTC 2020 [hello-local-device-pod] Hello from OpenEBS Local PV.
   </div>
   
## Troubleshooting 

Review the logs of the OpenEBS Local PV provisioner. OpenEBS Dynamic Local Provisioner logs can be fetched using. 

```
kubectl logs -n openebs -l openebs.io/component-name=openebs-localpv-provisioner
```

## Support

If you encounter issues or have a question, file an [Github issue](https://github.com/openebs/openebs/issues/new), or talk to us on the [#openebs channel on the Kubernetes Slack server](https://kubernetes.slack.com/messages/openebs/).

## See Also:

### [Understand OpenEBS Local PVs ](/docs/next/localpv.html)

### [Node Disk Manager](/docs/next/ugndm.html)


<br>


