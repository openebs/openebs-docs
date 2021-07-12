---
id: ugcstor-csi
title: cStor User Guide
sidebar_label: cStor
---
------
<br>
 
<a href="/docs/assets/svg/3-config-sequence.svg" target="_blank"><img src="/docs/assets/svg/3-config-sequence.svg" alt="OpenEBS configuration flow" style="width:100%"></a>
 
<br>
<br>
 
This user guide section provides the operations needed to be performed by the User and the Admin for configuring cStor related tasks.
 
  :::note
  The recommended approach to provision cStor Pools is to use CStorPoolCluster(CSPC), the detailed steps have been provided in this document. However, OpenEBS also supports provisioning of cStor Pools using StoragePoolClaim (SPC). For detailed instructions, refer to the <a href="https://docs.openebs.io/v260/docs/next/ugcstor.html" target="_blank">cStor User guide(SPC)</a>.<br>
  :::
 
 
 
<h2>Operations</h2>
 
[Creating cStor storage pools](#creating-cstor-storage-pool)
 
[Creating cStor storage classes](#creating-cstor-storage-classes)

[Deploying a sample application](#deploying-a-sample-application)

[Scaling up cStor pools](#scaling-cstor-pools)

[Snapshot and Clone of a cStor Volume](#snapshot-and-clone-of-a-cstor-volume)
 
[Expanding a cStor volume](#expanding-a-cstor-volume)
 
[Block Device Tagging](#block-device-tagging)
 
[Performance Tunings in cStor Pools](#performance-tunings-in-cstor-pools)
 
[Performance Tunings in cStor Volumes](#performance-tunings-in-cstor-volumes)

[Cleaning up a cStor setup](#cstor-cleanup)
 
 

 
## <a class="anchor" aria-hidden="true" id="user-operations"></a>Operations
 
### <a class="anchor" aria-hidden="true" id="creating-cstor-storage-pool"></a>Creating cStor storage pools
 
 
 <b>Prerequisites:</b>
 
- The latest release of OpenEBS cStor must be installed using cStor Operator yaml.
 
 ```
 kubectl apply -f https://openebs.github.io/charts/cstor-operator.yaml
 ```
      
 All the NDM cStor operator pods must be in a running state. To get the status of the pods execute:
 
 ```
 kubectl get pod -n openebs
 ```
 
 Sample Output:
 ```
 NAME                                             READY   STATUS    RESTARTS    AGE
 cspc-operator-5fb7db848f-wgnq8                    1/1    Running       0      6d7h
 cvc-operator-7f7d8dc4c5-sn7gv                     1/1    Running       0      6d7h
 openebs-cstor-admission-server-7585b9659b-rbkmn   1/1    Running       0      6d7h
 openebs-cstor-csi-controller-0                    6/6    Running       0      6d7h
 openebs-cstor-csi-node-dl58c                      2/2    Running       0      6d7h
 openebs-cstor-csi-node-jmpzv                      2/2    Running       0      6d7h
 openebs-cstor-csi-node-tfv45                      2/2    Running       0      6d7h
 openebs-ndm-gctb7                                 1/1    Running       0      6d7h
 openebs-ndm-operator-7c8759dbb5-58zpl             1/1    Running       0      6d7h
 openebs-ndm-sfczv                                 1/1    Running       0      6d7h
 openebs-ndm-vgdnv                                 1/1    Running       0      6d7h
 ```
 
- Nodes must have disks attached to them. To get the list of attached block devices, execute:
   ```
   kubectl get bd -n openebs
   ```
  Sample Output:
 
  ```
  NAME                                          NODENAME         SIZE         CLAIMSTATE  STATUS   AGE
  blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68  worker-node-3    21474836480   Unclaimed   Active   2m10s
  blockdevice-10ad9f484c299597ed1e126d7b857967  worker-node-1    21474836480   Unclaimed   Active   2m17s
  blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b  worker-node-2    21474836480   Unclaimed   Active   2m12s
 
  ```
 
<b>Creating a CStorPoolCluster:</b><br>
 
- Get all the node labels present in the cluster with the following command, these node labels will be required to modify the CSPC yaml.
  ```
  kubectl get node --show-labels
  ```
 Sample Output:
  ```
  NAME               STATUS   ROLES    AGE    VERSION   LABELS
 
  master             Ready    master   5d2h   v1.20.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=master,kubernetes.io/os=linux,node-role.kubernetes.io/master=
 
  worker-node-1      Ready    <none>   5d2h   v1.20.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker-node-1,kubernetes.io/os=linux
 
  worker-node-2      Ready    <none>   5d2h   v1.20.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker-node-2,kubernetes.io/os=linux
 
  worker-node-3      Ready    <none>   5d2h   v1.18.0   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,kubernetes.io/hostname=worker-node-3,kubernetes.io/os=linux
  ```
 
- Modify the CSPC yaml to use the worker nodes. Use the value from labels kubernetes.io/hostname=&lt; node_name &gt;. This label value and node name could be different in some platforms. In this case, the label values and node names are:
  <code>kubernetes.io/hostname:"worker-node-1"</code>, <code>kubernetes.io/hostname: "worker-node-2"</code> and <code>kubernetes.io/hostname: "worker-node-3"</code>.
- Modify CSPC yaml file to add a block device attached to the same node where the pool is to be provisioned. <br><br>
**Note:** The <code>dataRaidGroupType:</code> can either be set as <code>stripe</code> or <code>mirror</code> as per your requirement. In the following example it is configured as <code>stripe</code>.<br><br>
 Sample CSPC yaml:
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
     poolConfig:
       dataRaidGroupType: "stripe"
```
We have named the configuration YAML file as <code>cspc.yaml</code>. Execute the following command for CSPC creation,
 
```
kubectl apply -f cspc.yaml
```
To verify the status of created CSPC, execute:
```
kubectl get cspc -n openebs
```
Sample Output:
```
NAME                   HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES     AGE
cstor-disk-pool        3                  3                      3                    2m2s
```
Check if the pool instances report their status as <b>ONLINE</b> using the below command:
 
```
kubectl get cspi -n openebs
```
Sample Output:
 
```
NAME                  HOSTNAME             ALLOCATED   FREE    CAPACITY   STATUS   AGE
cstor-disk-pool-vn92  worker-node-1        60k         9900M    9900M     ONLINE   2m17s
cstor-disk-pool-al65  worker-node-2        60k         9900M    9900M     ONLINE   2m17s
cstor-disk-pool-y7pn  worker-node-3        60k         9900M    9900M     ONLINE   2m17s
```
Once all the pods are in running state, these pool instances can be used for creation of cStor volumes.
<hr>
 
### <a class="anchor" aria-hidden="true" id="creating-cstor-storage-classes"></a>Creating cStor storage classes
 
StorageClass definition is an important task in the planning and execution of OpenEBS storage. The real power of CAS architecture is to give an independent or a dedicated storage engine like cStor for each workload, so that granular policies can be applied to that storage engine to tune the behaviour or performance as per the workload's need.
  #### Steps to create a cStor StorageClass:
  1. Decide the CStorPoolCluster for which you want to create a Storage Class.
  2. Decide the replicaCount based on your requirement/workloads. OpenEBS doesn't restrict the replica count to set, but a <b>maximum of 5</b> replicas are allowed. It depends how users configure it, but for the availability of volumes <b>at least (n/2 + 1) replicas</b> should be up and connected to the target, where n is the replicaCount. The Replica Count should be always less  than or equal to the number of cStor Pool Instances(CSPIs). The following are some example cases:
   <ul>
   <li>If a user configured replica count as 2, then always 2 replicas should be available to perform operations on volume.</li>
   <li>If a user configured replica count as 3 it should require at least 2 replicas should be available for volume to be operational.</li>
    <li>If a user configured replica count as 5 it should require at least 3 replicas should be available for volume to be operational.</li>
   </ul>
  3. Create a YAML spec file <code>cstor-csi-disk.yaml</code> using the template given below. Update the pool, replica count and other policies. By using this sample configuration YAML, a StorageClass will be created with 3 OpenEBS cStor replicas and will configure themselves on the pool instances.
 
     ```
     kind: StorageClass
     apiVersion: storage.k8s.io/v1
     metadata:
       name: cstor-csi-disk
     provisioner: cstor.csi.openebs.io
     allowVolumeExpansion: true
     parameters:
       cas-type: cstor
       # cstorPoolCluster should have the name of the CSPC
       cstorPoolCluster: cstor-disk-pool
       # replicaCount should be <= no. of CSPI
       replicaCount: "3"
     ```
To deploy the YAML, execute:
```
kubectl apply -f cstor-csi-disk.yaml
```
To verify, execute:
 
```
kubectl get sc
```
Sample Output:
```
NAME                        PROVISIONER                                                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
cstor-csi                   cstor.csi.openebs.io                                       Delete          Immediate              true                   4s
```
<hr>

 ### <a class="anchor" aria-hidden="true" id="deploying-a-sample-application"></a>Deploying a sample application

To deploy a sample application using the above created CSPC and StorageClass, a PVC, that utilises the created StorageClass, needs to be deployed. Given below is an example YAML for a PVC which uses the SC created earlier.

```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cstor-pvc
spec:
  storageClassName: cstor-csi-disk
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
```
Apply the above PVC yaml to dynamically create volume and verify that the PVC has been successfully created and bound to a PersistentVolume (PV).

```
kubectl get pvc
```
Sample Output:
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
cstor-pvc        Bound    pvc-f1383b36-2d4d-4e9f-9082-6728d6c55bd1   5Gi        RWO            cstor-csi-disk   12s
```
Now, to deploy an application using the above created PVC specify the <code>claimName</code> parameter under <code>volumes</code>. 
<br>
Given below is a sample busybox application YAML that uses the PVC created earlier.
```
apiVersion: v1
kind: Pod
metadata:
  name: busybox
  namespace: default
spec:
  containers:
  - command:
       - sh
       - -c
       - 'date >> /mnt/openebs-csi/date.txt; hostname >> /mnt/openebs-csi/hostname.txt; sync; sleep 5; sync; tail -f /dev/null;'
    image: busybox
    imagePullPolicy: Always
    name: busybox
    volumeMounts:
    - mountPath: /mnt/openebs-csi
      name: demo-vol
  volumes:
  - name: demo-vol
    persistentVolumeClaim:
      claimName: cstor-pvc
```
Apply the above YAML.
Verify that the pod is running and is able to write data to the volume.
```
kubectl get pods
```
Sample Output:
```
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          97s
```
The example busybox application will write the current date into the mounted path, i.e, <i>/mnt/openebs-csi/date.txt</i> when it starts. To verify, exec into the busybox container.
```
kubectl exec -it busybox -- cat /mnt/openebs-csi/date.txt
```
Sample output:
```
Fri May 28 05:00:31 UTC 2021
```
<hr>
 
### <a class="anchor" aria-hidden="true" id="scaling-cstor-pools"></a>Scaling cStor pools

Once the cStor storage pools are created you can scale-up your existing cStor pool.
To scale-up the pool size, you need to edit the CSPC YAML that was used for creation of CStorPoolCluster.
 
Scaling up can done by two methods:
1. [Adding new nodes(with new disks) to the existing CSPC](#adding-disk-new-node)
2. [Adding new disks to existing nodes](#adding-disk-same-node)

**Note:** The dataRaidGroupType: can either be set as stripe or mirror as per your requirement. In the following example it is configured as stripe.

#### <a class="anchor" aria-hidden="true" id="adding-disk-new-node"></a>Adding new nodes(with new disks) to the existing CSPC
A new node spec needs to be added to previously deployed YAML,
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
     poolConfig:
       dataRaidGroupType: "stripe"
 
   # New node spec added -- to create a cStor pool on worker-3
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-4"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-02d9b2dc8954ce0347850b7625375e24"
     poolConfig:
       dataRaidGroupType: "stripe"
 
```
Now verify the status of CSPC and CSPI(s):
 
```
kubectl get cspc -n openebs
```
 
Sample Output:
```
NAME                     HEALTHYINSTANCES   PROVISIONEDINSTANCES   DESIREDINSTANCES   AGE
cspc-disk-pool           4                  4                      4                  8m5s
```
 
```
kubectl get cspi -n openebs
```
 
```
NAME                  HOSTNAME         FREE     CAPACITY    READONLY   STATUS   AGE
cspc-disk-pool-d9zf   worker-node-1    28800M   28800071k   false      ONLINE   7m50s
cspc-disk-pool-lr6z   worker-node-2    28800M   28800056k   false      ONLINE   7m50s
cspc-disk-pool-x4b4   worker-node-3    28800M   28800056k   false      ONLINE   7m50s
cspc-disk-pool-rt4k   worker-node-4    28800M   28800056k   false      ONLINE   15s
 
```
As a result of this, we can see that a new pool have been added, increasing the number of pools to 4

#### <a class="anchor" aria-hidden="true" id="adding-disk-same-node"></a>Adding new disks to existing nodes
A new <code>blockDeviceName</code> under <code>blockDevices</code> needs to be added to previously deployed YAML. Execute the following command to edit the CSPC,
```
kubectl edit cspc -n openebs cstor-disk-pool
```
Sample YAML:
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-10ad9f484c299597ed1e126d7b857967"
           - blockDeviceName: "blockdevice-f036513d98f6c7ce31fd6e1ac3fad2f5" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b"
           - blockDeviceName: "blockdevice-fb7c995c4beccd6c872b7b77aad32932" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"
 
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68"
           - blockDeviceName: "blockdevice-46ddda7223b35b81415b0a1b12e40bcb" //# New blockdevice added
     poolConfig:
       dataRaidGroupType: "stripe"
 
```

<hr>

### <a class="anchor" aria-hidden="true" id="snapshot-and-clone-of-a-cstor-volume"></a>Snapshot and Clone of a cStor Volume
 
An OpenEBS snapshot is a set of reference markers for data at a particular point in time. A snapshot act as a detailed table of contents, with accessible copies of data that user can roll back to the required point of instance. Snapshots in OpenEBS are instantaneous and are managed through kubectl.
 
During the installation of OpenEBS, a snapshot-controller and a snapshot-provisioner are setup which assist in taking the snapshots. During the snapshot creation, snapshot-controller creates VolumeSnapshot and VolumeSnapshotData custom resources. A snapshot-provisioner is used to restore a snapshot as a new Persistent Volume(PV) via dynamic provisioning.
 #### Creating a cStor volume Snapshot
 
 1. Before proceeding to create a cStor volume snapshot and use it further for restoration, it is necessary to create a <code>VolumeSnapshotClass</code>. Copy the following YAML specification into a file called <code>snapshot_class.yaml</code>.
```
kind: VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
metadata:
 name: csi-cstor-snapshotclass
 annotations:
   snapshot.storage.kubernetes.io/is-default-class: "true"
driver: cstor.csi.openebs.io
deletionPolicy: Delete
```
The deletion policy can be set as <code>Delete or Retain</code>. When it is set to Retain, the underlying physical snapshot on the storage cluster is retained even when the VolumeSnapshot object is deleted.
To apply, execute:
```
kubectl apply -f snapshot_class.yaml
```
 
**Note:** In clusters that only install <code>v1beta1</code> version of VolumeSnapshotClass as the supported version(eg. OpenShift(OCP) 4.5 ), the following error might be encountered.
```
no matches for kind "VolumeSnapshotClass" in version "snapshot.storage.k8s.io/v1"
```
In such cases, the apiVersion needs to be updated to <code>apiVersion: snapshot.storage.k8s.io/v1beta1</code>
 
2.  For creating the snapshot, you need to create a YAML specification and provide the required PVC name into it. The only prerequisite check is to be performed is to ensure that there is no stale entries of snapshot and snapshot data before creating a new snapshot. Copy the following YAML specification into a file called <code>snapshot.yaml</code>.
```
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
 name: cstor-pvc-snap
spec:
 volumeSnapshotClassName: csi-cstor-snapshotclass
 source:
   persistentVolumeClaimName: cstor-pvc
```
Run the following command to create the snapshot,
```
kubectl create -f snapshot.yaml
```
To list the snapshots, execute:
```
kubectl get volumesnapshots -n default
```
Sample Output:
```
NAME                        AGE
cstor-pvc-snap              10s
```
A VolumeSnapshot is analogous to a PVC and is associated with a <code>VolumeSnapshotContent</code> object that represents the actual snapshot. To identify the VolumeSnapshotContent object for the VolumeSnapshot execute:
 
```
kubectl describe volumesnapshots cstor-pvc-snap -n default
```
Sample Output:
```
Name:         cstor-pvc-snap
Namespace:    default
.
.
.
Spec:
 Snapshot Class Name:    cstor-csi-snapshotclass
 Snapshot Content Name:  snapcontent-e8d8a0ca-9826-11e9-9807-525400f3f660
 Source:
   API Group:
   Kind:       PersistentVolumeClaim
   Name:       cstor-pvc
Status:
 Creation Time:  2020-06-20T15:27:29Z
 Ready To Use:   true
 Restore Size:   5Gi
 
```
 
The <code>SnapshotContentName</code> identifies the <code>VolumeSnapshotContent</code> object which serves this snapshot. The <code>Ready To Use</code> parameter indicates that the Snapshot has been created successfully and can be used to create a new PVC.
 
**Note:** All cStor snapshots should be created in the same namespace of source PVC.
#### Cloning a cStor Snapshot
 
Once the snapshot is created, you can use it to create a PVC. In order to restore a specific snapshot, you need to create a new PVC that refers to the snapshot. Below is an example of a YAML file that restores and creates a PVC from a snapshot.
 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: restore-cstor-pvc
spec:
 storageClassName: cstor-csi-disk
 dataSource:
   name: cstor-pvc-snap
   kind: VolumeSnapshot
   apiGroup: snapshot.storage.k8s.io
 accessModes:
   - ReadWriteOnce
 resources:
   requests:
     storage: 5Gi
```
The <code>dataSource</code> shows that the PVC must be created using a VolumeSnapshot named <code>cstor-pvc-snap</code> as the source of the data. This instructs cStor CSI to create a PVC from the snapshot. Once the PVC is created, it can be attached to a pod and used just like any other PVC.
 
To verify the creation of PVC execute:
```
kubectl get pvc
```
Sample Output:
```
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
restore-cstor-pvc              Bound    pvc-2f2d65fc-0784-11ea-b887-42010a80006c   5Gi        RWO            cstor-csi-disk            5s
```
<hr>

### <a class="anchor" aria-hidden="true" id="expanding-a-cstor-volume"></a>Expanding a cStor volume
 
OpenEBS cStor introduces support for expanding a PersistentVolume using the CSI provisioner. Provided cStor is configured to function as a CSI provisioner, you can expand PVs that have been created by cStor CSI Driver. This feature is supported with Kubernetes versions 1.16 and above.
<br>
For expanding a cStor PV, you must ensure the following items are taken care of:
<br>
<ul>
<li>The StorageClass must support volume expansion. This can be done by editing the StorageClass definition to set the allowVolumeExpansion: true.</li>
<li>To resize a PV, edit the PVC definition and update the spec.resources.requests.storage to reflect the newly desired size, which must be greater than the original size.</li>
<li>The PV must be attached to a pod for it to be resized. There are two scenarios when resizing an cStor PV:<br>
<ul>
   <li>If the PV is attached to a pod, cStor CSI driver expands the volume on the storage backend, re-scans the device and resizes the filesystem.</li>
   <li>When attempting to resize an unattached PV, cStor CSI driver expands the volume on the storage backend. Once the PVC is bound to a pod, the driver re-scans the device and resizes the filesystem. Kubernetes then updates the PVC size after the expansion operation has successfully completed.</li>
</ul>
</ul>
<br>
Below example shows the way for expanding cStor volume and how it works. For an already existing StorageClass, you can edit the StorageClass to include the <code>allowVolumeExpansion: true</code>  parameter.
 
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
 name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
 replicaCount: "3"
 cstorPoolCluster: "cspc-disk-pool"
 cas-type: "cstor"
```
 
For example an application busybox pod is using the below PVC associated with PV. To get the status of the pod, execute:
 
```
$ kubectl get pods
```
 
The following is a sample output:
```
NAME            READY   STATUS    RESTARTS   AGE
busybox         1/1     Running   0          38m
```
 
To list PVCs, execute:
 
```
$ kubectl get pvc
```
Sample Output:
```
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
cstor-pvc                      Bound    pvc-849bd646-6d3f-4a87-909e-2416d4e00904   5Gi        RWO            cstor-csi-disk            1d
```
 
To list PVs, execute:
 
```
$ kubectl get pv
```
 
Sample Output:
 
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS        REASON   AGE
pvc-849bd646-6d3f-4a87-909e-2416d4e00904   5Gi        RWO            Delete           Bound    default/cstor-pvc       cstor-csi-disk               40m
```
 
<br>
To resize the PV that has been created from 5Gi to 10Gi, edit the PVC definition and update the spec.resources.requests.storage to 10Gi. It may take a few seconds to update the actual size in the PVC resource, wait for the updated capacity to reflect in PVC status (pvc.status.capacity.storage). It is internally a two step process for volumes containing a file system:
<br>
<ul>
<li>Volume expansion</li>
<li>FileSystem expansion</li>
</ul>
 
```
$ kubectl edit pvc cstor-pvc
```
 
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 annotations:
   pv.kubernetes.io/bind-completed: "yes"
   pv.kubernetes.io/bound-by-controller: "yes"
   volume.beta.kubernetes.io/storage-provisioner: cstor.csi.openebs.io
 creationTimestamp: "2020-06-24T12:22:24Z"
 finalizers:
 - kubernetes.io/pvc-protection
   name: cstor-pvc
 namespace: default
 resourceVersion: "766"
 selfLink: /api/v1/namespaces/default/persistentvolumeclaims/cstor-pvc
 uid: 849bd646-6d3f-4a87-909e-2416d4e00904
spec:
 accessModes:
 - ReadWriteOnce
 resources:
   requests:
     storage: 10Gi
```
Now, we can validate the resize has worked correctly by checking the size of the PVC, PV, or describing the pvc to get all events.
 
```
$ kubectl describe pvc cstor-pvc
```
```
Name:          cstor-pvc
Namespace:     default
StorageClass:  cstor-csi-disk
Status:        Bound
Volume:        pvc-849bd646-6d3f-4a87-909e-2416d4e00904
Labels:        <none>
Annotations:   pv.kubernetes.io/bind-completed: yes
              pv.kubernetes.io/bound-by-controller: yes
              volume.beta.kubernetes.io/storage-provisioner: cstor.csi.openebs.io
Finalizers:    [kubernetes.io/pvc-protection]
Capacity:      10Gi
Access Modes:  RWO
VolumeMode:    Filesystem
Mounted By:    busybox-cstor
Events:
 Type     Reason                      Age                From                                                                                      Message
 ----     ------                      ----               ----                                                                                      -------
 Normal   ExternalProvisioning        46m (x2 over 46m)  persistentvolume-controller                                                               waiting for a volume to be created, either by external provisioner "cstor.csi.openebs.io" or manually created by system administrator
 Normal   Provisioning                46m                cstor.csi.openebs.io_openebs-cstor-csi-controller-0_bcba3893-c1c4-4e86-aee4-de98858ec0b7  External provisioner is provisioning volume for claim "default/claim-csi-123"
 Normal   ProvisioningSucceeded       46m                cstor.csi.openebs.io_openebs-cstor-csi-controller-0_bcba3893-c1c4-4e86-aee4-de98858ec0b7  Successfully provisioned volume pvc-849bd646-6d3f-4a87-909e-2416d4e00904
 Warning  ExternalExpanding           93s                volume_expand                                                                             Ignoring the PVC: didn't find a plugin capable of expanding the volume; waiting for an external controller to process this PVC.
 Normal   Resizing                    93s                external-resizer cstor.csi.openebs.io                                                     External resizer is resizing volume pvc-849bd646-6d3f-4a87-909e-2416d4e00904
 Normal   FileSystemResizeRequired    88s                external-resizer cstor.csi.openebs.io                                                     Require file system resize of volume on node
 Normal   FileSystemResizeSuccessful  4s                 kubelet, 127.0.0.1                                                                        MountVolume.NodeExpandVolume succeeded for volume "pvc-849bd646-6d3f-4a87-909e-2416d4e00904"
```
 
```
$ kubectl get pvc
```
 
Sample Output:
```
NAME                           STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS              AGE
cstor-pvc                      Bound    pvc-849bd646-6d3f-4a87-909e-2416d4e00904   10Gi        RWO            cstor-csi-disk            1d
```
 
```
$ kubectl get pv
```
 
Sample Output:
 
```
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                   STORAGECLASS        REASON   AGE
pvc-849bd646-6d3f-4a87-909e-2416d4e00904   10Gi        RWO            Delete           Bound    default/cstor-pvc       cstor-csi-disk               40m
```
<hr>
 
### <a class="anchor" aria-hidden="true" id="block-device-tagging"></a>Block Device Tagging
NDM provides you with an ability to reserve block devices to be used for specific applications via adding tag(s) to your block device(s). This feature can be used by cStor operators to specify the block devices which should be consumed by cStor pools and conversely restrict anyone else from using those block devices. This helps in protecting against manual errors in specifying the block devices in the CSPC yaml by users.
1. Consider the following block devices in a Kubernetes cluster, they will be used to provision a  storage pool. List the labels added to these block devices,
 
  ```
  kubectl get bd -n openebs --show-labels
  ```
  Sample Output:
 
  ```
  NAME                                           NODENAME               SIZE          CLAIMSTATE   STATUS   AGE   LABELS
  blockdevice-00439dc464b785256242113bf0ef64b9   worker-node-3          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-3,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true
  blockdevice-022674b5f97f06195fe962a7a61fcb64   worker-node-1          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-1,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true
  blockdevice-241fb162b8d0eafc640ed89588a832df   worker-node-2          21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-2,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true
 
  ```
2. Now, to understand how block device tagging works we will be adding <code>openebs.io/block-device-tag=fast</code> to the block device attached to worker-node-3 <i>(i.e blockdevice-00439dc464b785256242113bf0ef64b9)</i>
 
  ```
  kubectl label bd blockdevice-00439dc464b785256242113bf0ef64b9 -n openebs  openebs.io/block-device-tag=fast
  ```
 
  ```
  kubectl get bd -n openebs blockdevice-00439dc464b785256242113bf0ef64b9 --show-labels
  ```
  Sample Output:
  ```
  NAME                                           NODENAME             SIZE          CLAIMSTATE   STATUS   AGE   LABELS
  blockdevice-00439dc464b785256242113bf0ef64b9   worker-node-3        21473771008   Unclaimed    Active   34h   kubernetes.io/hostname=worker-node-3,ndm.io/blockdevice-type=blockdevice,ndm.io/managed=true,openebs.io/block-device-tag=fast
  ```
  Now, provision cStor pools using the following CSPC YAML. Note, <code>openebs.io/allowed-bd-tags:</code> is set to <code>cstor,ssd</code> which ensures the CSPC will be created using the block devices that either have the label set to cstor or ssd, or have no such label.
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cspc-disk-pool
 namespace: openebs
 annotations:
  # This annotation helps to specify the BD that can be allowed.
  openebs.io/allowed-bd-tags: cstor,ssd
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: "worker-node-1"
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: "blockdevice-022674b5f97f06195fe962a7a61fcb64"
     poolConfig:
       dataRaidGroupType: "stripe"
- nodeSelector:
       kubernetes.io/hostname: "worker-node-2"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-241fb162b8d0eafc640ed89588a832df"
     poolConfig:
       dataRaidGroupType: "stripe"
- nodeSelector:
       kubernetes.io/hostname: "worker-node-3"
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: "blockdevice-00439dc464b785256242113bf0ef64b9"
     poolConfig:
       dataRaidGroupType: "stripe"
```
Apply the above CSPC file for CSPIs to get created and check the CSPI status.
 
```
kubectl apply -f cspc.yaml
```
 
```
kubectl get cspi -n openebs
```
Sample Output:
 
```
NAME             HOSTNAME        FREE   CAPACITY    READONLY PROVISIONEDREPLICAS HEALTHYREPLICAS STATUS   AGE
cspc-stripe-b9f6 worker-node-2   19300M 19300614k   false      0                     0          ONLINE   89s
cspc-stripe-q7xn worker-node-1   19300M 19300614k   false      0                     0          ONLINE   89s
 
```
Note that CSPI for node <b>worker-node-3</b> is not created because:
 - CSPC YAML created above has <code>openebs.io/allowed-bd-tags: cstor,ssd</code> in its annotation. Which means that the CSPC operator will only consider those block devices for provisioning that either do not have a BD tag, openebs.io/block-device-tag, on the block device or have the tag with the values set as <code>cstor or ssd</code>.
 - In this case, the blockdevice-022674b5f97f06195fe962a7a61fcb64 (on node worker-node-1) and blockdevice-241fb162b8d0eafc640ed89588a832df (on node worker-node-2) do not have the label. Hence, no restrictions are applied on it and they can be used as the CSPC operator for pool provisioning.
 - For blockdevice-00439dc464b785256242113bf0ef64b9 (on node worker-node-3), the label <code>openebs.io/block-device-tag</code> has the value fast. But on the CSPC, the annotation openebs.io/allowed-bd-tags has value cstor and ssd. There is no fast keyword present in the annotation value and hence this block device cannot be used.
 
**NOTE:**
1. To allow multiple tag values, the bd tag annotation can be written in the following comma-separated manner:
  ```
  openebs.io/allowed-bd-tags: fast,ssd,nvme
  ```
2. BD tag can only have one value on the block device CR. For example,
   - openebs.io/block-device-tag: fast
   Block devices should not be tagged in a comma-separated format. One of the reasons for this is, cStor allowed bd tag annotation takes comma-separated values and values like(i.e fast,ssd ) can never be interpreted as a single word in cStor and hence BDs tagged in above format cannot be utilised by cStor.
3. If any block device mentioned in CSPC has an empty value for <code>the openebs.io/block-device-tag</code>, then those block devices will not be considered for pool provisioning and other operations. Block devices with empty tag value are implicitly not allowed by the CSPC operator.
 
 
<hr>


### <a class="anchor" aria-hidden="true" id="performance-tunings-in-cstor-pools"></a>Performance Tunings in cStor Pools
 
Allow users to set available performance tunings in cStor Pools based on their workload. cStor pool(s) can be tuned via CSPC and is the recommended way to do it. Below are the tunings that can be applied:
<br>
**Resource requests and limits:** This ensures high quality of service when applied for the pool manager containers. 
 
**Toleration for pool manager pod:** This ensures scheduling of pool pods on the tainted nodes.
 
**Set priority class:** Sets the priority levels as required.
 
**Compression:** This helps in setting the compression for cStor pools.
 
**ReadOnly threshold:** Helps in specifying read only thresholds for cStor pools.
<br>
 
<font size="4">**Example configuration for Resource and Limits:**</font>
 
Following CSPC YAML specifies resources and auxResources that will get applied to all pool manager pods for the CSPC. Resources get applied to cstor-pool containers and auxResources gets applied to sidecar containers i.e. cstor-pool-mgmt and pool-exporter.
<br>
In the following CSPC YAML we have only one pool spec (@spec.pools). It is also possible to override the resource and limit value for a specific pool.
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 resources:
   requests:
     memory: "2Gi"
     cpu: "250m"
   limits:
     memory: "4Gi"
     cpu: "500m"
 
 auxResources:
   requests:
     memory: "500Mi"
     cpu: "100m"
   limits:
     memory: "1Gi"
     cpu: "200m"
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
 
     poolConfig:
       dataRaidGroupType: mirror
```
 
Following CSPC YAML explains how the resource and limits can be overridden. If you look at the CSPC YAML, there are no resources and auxResources specified at pool level for worker-node-1 and worker-node-2 but specified for worker-node-3. In this case, for worker-node-1 and worker-node-2 the resources and auxResources will be applied from @spec.resources and @spec.auxResources respectively but for worker-node-3 these will be applied from @spec.pools[2].poolConfig.resources and @spec.pools[2].poolConfig.auxResources respectively.
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 resources:
   requests:
     memory: "64Mi"
     cpu: "250m"
   limits:
     memory: "128Mi"
     cpu: "500m"
 auxResources:
   requests:
     memory: "50Mi"
     cpu: "400m"
   limits:
     memory: "100Mi"
     cpu: "400m"
  pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
     poolConfig:
       dataRaidGroupType: mirror
  
   - nodeSelector:
       kubernetes.io/hostname: worker-node-2
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40
     poolConfig:
       dataRaidGroupType: mirror
  
   - nodeSelector:
       kubernetes.io/hostname: worker-node-3
     dataRaidGroups:
       - blockDevices:
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
           - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43
     poolConfig:
       dataRaidGroupType: mirror
       resources:
         requests:
           memory: 70Mi
           cpu: 300m
         limits:
           memory: 130Mi
           cpu: 600m
       auxResources:
         requests:
           memory: 60Mi
           cpu: 500m
         limits:
           memory: 120Mi
           cpu: 500m
 
```
 
<font size="4">**Example configuration for Tolerations:**</font>
 
Tolerations are applied in a similar manner like resources and auxResources. The following is a sample CSPC YAML that has tolerations specified. For worker-node-1 and worker-node-2 tolerations are applied form @spec.tolerations but for worker-node-3 it is applied from @spec.pools[2].poolConfig.tolerations
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 
 tolerations:
 - key: data-plane-node
   operator: Equal
   value: true
   effect: NoSchedule
 
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
 
     poolConfig:
       dataRaidGroupType: mirror
 
   - nodeSelector:
       kubernetes.io/hostname: worker-node-2
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40
 
     poolConfig:
       dataRaidGroupType: mirror
 
   - nodeSelector:
       kubernetes.io/hostname: worker-node-3
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43
 
     poolConfig:
       dataRaidGroupType: mirror
       tolerations:
       - key: data-plane-node
         operator: Equal
         value: true
         effect: NoSchedule
 
       - key: apac-zone
         operator: Equal
         value: true
         effect: NoSchedule
```
 
<font size="4">**Example configuration for Priority Class:**</font>
 
Priority Classes are also applied in a similar manner like resources and auxResources. The following is a sample CSPC YAML that has a priority class specified. For worker-node-1 and worker-node-2 priority classes are applied from @spec.priorityClassName but for worker-node-3 it is applied from @spec.pools[2].poolConfig.priorityClassName. Check more info about [priorityclass](https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass).
 
**Note:**
<ol>
<li>Priority class needs to be created beforehand. In this case, high-priority and ultra-priority priority classes should exist.</li>
<li>The index starts from 0 for @.spec.pools list.</li>
</ol>
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 
 priorityClassName: high-priority
 
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
 
     poolConfig:
       dataRaidGroupType: mirror
 
   - nodeSelector:
       kubernetes.io/hostname: worker-node-2
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f39
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f40
 
     poolConfig:
       dataRaidGroupType: mirror
 
   - nodeSelector:
       kubernetes.io/hostname: worker-node-3
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f42
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f43
 
     poolConfig:
       dataRaidGroupType: mirror
       priorityClassName: utlra-priority
```
<font size="4">**Example configuration for Compression:**</font>
 
Compression values can be set at <b>pool level only</b>. There is no override mechanism like it was there in case of tolerations, resources, auxResources and priorityClass. Compression value must be one of
- on
- off
- lzjb
- gzip
- gzip-[1-9]
- zle
- lz4
 
**Note:** lz4 is the default compression algorithm that is used if the compression field is left unspecified on the cspc. Below is the sample yaml which has compression specified.
 
```
 
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-disk-pool
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
 
     poolConfig:
       dataRaidGroupType: mirror
       compression: lz4
```
<font size="4">**Example configuration for Read Only Threshold:**</font>
 
RO threshold can be set in a similar manner like compression. ROThresholdLimit is the threshold(percentage base) limit for pool read only mode. If ROThresholdLimit(%) amount of pool storage is consumed then the pool will be set to readonly. If ROThresholdLimit is set to 100 then entire pool storage will be used. By default it will be set to 85% i.e when unspecified on the CSPC.ROThresholdLimit value will be 0 < ROThresholdLimit <= 100. Following CSPC yaml has the ReadOnly Threshold percentage specified.
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorPoolCluster
metadata:
 name: cstor-csi-disk
 namespace: openebs
spec:
 pools:
   - nodeSelector:
       kubernetes.io/hostname: worker-node-1
 
     dataRaidGroups:
     - blockDevices:
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f36
         - blockDeviceName: blockdevice-ada8ef910929513c1ad650c08fbe3f37
 
     poolConfig:
       dataRaidGroupType: mirror
 
       roThresholdLimit : 70
```
<hr>
 
### <a class="anchor" aria-hidden="true" id="performance-tunings-in-cstor-volumes"></a>Performance Tunings in cStor Volumes
 
Similar to tuning of the cStor Pool cluster, there are possible ways for tuning cStor volumes. cStor volumes can be provisioned using different policy configurations. However, <code>cStorVolumePolicy</code> needs to be created first. It must be created prior to creation of StorageClass as  <code>CStorVolumePolicy</code> name needs to be specified to provision cStor volume based on configured policy. A sample StorageClass YAML that utilises <code>cstorVolumePolicy</code> is given below for reference:<br>
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
 name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
parameters:
 replicaCount: "1"
 cstorPoolCluster: "cstor-disk-pool"
 cas-type: "cstor"
 fsType: "xfs"                 // default type is ext4
 cstorVolumePolicy: "csi-volume-policy"
```
 
If the volume policy is not created before volume provisioning and needs to be modified later,
it can be changed by editing the cStorVolumeConfig(CVC) resource as per volume bases which will be reconciled by the CVC controller to the respected volume resources.
Each PVC creation request will create a CStorVolumeConfig(cvc) resource which can be used to manage volume, its policies and any supported operations (like, Scale up/down), per volume bases.
To edit, execute:

```
kubectl edit cvc <pv-name> -n openebs
```
Sample Output:

```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumeConfig
metadata:
  annotations:
    openebs.io/persistent-volume-claim: "cstor-pvc"
    openebs.io/volume-policy: csi-volume-policy
    openebs.io/volumeID: pvc-25e79ecb-8357-49d4-83c2-2e63ebd66278
  creationTimestamp: "2020-07-22T11:36:13Z"
  finalizers:
  - cvc.openebs.io/finalizer
  generation: 3
  labels:
    cstor.openebs.io/template-hash: "3278395555"
    openebs.io/cstor-pool-cluster: cstor-disk-pool
  name: pvc-25e79ecb-8357-49d4-83c2-2e63ebd66278
  namespace: openebs
  resourceVersion: "1283"
  selfLink: /apis/cstor.openebs.io/v1/namespaces/openebs/cstorvolumeconfigs/pvc-25e79ecb-8357-49d4-83c2-2e63ebd66278
  uid: 389320d8-5f0b-439d-8ef2-59f4d01b393a
publish:
  nodeId: 127.0.0.1
spec:
  capacity:
    storage: 1Gi
  cstorVolumeRef:
    apiVersion: cstor.openebs.io/v1
    kind: CStorVolume
    name: pvc-25e79ecb-8357-49d4-83c2-2e63ebd66278
    namespace: openebs
    resourceVersion: "1260"
    uid: ea6e09f2-1e65-41ab-820a-ed1ecd14873c
  policy:
    provision:
      replicaAffinity: true
    replica:
      zvolWorkers: "1"
    replicaPoolInfo:
    - poolName: cstor-disk-pool-vn92
    target:
      affinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: openebs.io/target-affinity
              operator: In
              values:
              - percona
          namespaces:
          - default
          topologyKey: kubernetes.io/hostname
      auxResources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
          cpu: 250m
          memory: 64Mi
      luWorkers: 8
      priorityClassName: system-cluster-critical
      queueDepth: "16"
      resources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
        .
        .
        .
```
 
 
The list of policies that can be configured are as follows:
- [Replica Affinity to create a volume replica on specific pool](#replica-affinity)
 
- [Volume Target Pod Affinity](#volume-target-pod-affinity)
 
- [Volume Tunable](#volume-tunable)
 
- [Memory and CPU Resources QoS](#memory-and-cpu-qos)
 
- [Toleration for target pod to ensure scheduling of target pods on tainted nodes](#toleration-for-target-pod)
 
- [Priority class for volume target deployment](#priority-class-for-volume-target-deployment)

 
 
#### <a class="anchor" aria-hidden="true" id="replica-affinity"></a>Replica Affinity to create a volume replica on specific pool
 
For StatefulSet applications, to distribute single replica volume on specific cStor pool we can use replicaAffinity enabled scheduling. This feature should be used with delay volume binding i.e. <code>volumeBindingMode: WaitForFirstConsumer</code> in StorageClass. When <code>volumeBindingMode</code> is set to <code>WaitForFirstConsumer</code> the csi-provisioner waits for the scheduler to select a node. The topology of the selected node will then be set as the first entry in preferred list and will be used by the volume controller to create the volume replica on the cstor pool scheduled on preferred node.
```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
 name: cstor-csi-disk
provisioner: cstor.csi.openebs.io
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
parameters:
 replicaCount: "1"
 cstorPoolCluster: "cstor-disk-pool"
 cas-type: "cstor"
 cstorVolumePolicy: "csi-volume-policy" // policy created with replicaAffinity set to true
```
 
The <code>replicaAffinity</code> spec needs to be enabled via volume policy before provisioning the volume
 
```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
 name: csi-volume-policy
 namespace: openebs
spec:
 provision:
   replicaAffinity: true
```
 
 
#### <a class="anchor" aria-hidden="true" id="volume-target-pod-affinity"></a>Volume Target Pod Affinity
The Stateful workloads access the OpenEBS storage volume by connecting to the Volume Target Pod. Target Pod Affinity policy can be used to co-locate volume target pod on the same node as the workload. This feature makes use of the Kubernetes Pod Affinity feature that is dependent on the Pod labels.
For this labels need to be added  to both, Application and volume Policy.
Given below is a sample YAML of <code>CStorVolumePolicy</code> having target-affinity label using <code>kubernetes.io/hostname</code> as a topologyKey in CStorVolumePolicy:
```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
  name: csi-volume-policy
  namespace: openebs
spec:
  target:
    affinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: openebs.io/target-affinity
            operator: In
            values:
            - fio-cstor                              // application-unique-label
        topologyKey: kubernetes.io/hostname
        namespaces: ["default"]                      // application namespace
```

Set the label configured in volume policy, openebs.io/target-affinity: fio-cstor ,on the app pod which will be used to find pods, by label, within the domain defined by topologyKey.

```
apiVersion: v1
kind: Pod
metadata:
  name: fio-cstor
  namespace: default
  labels:
    name: fio-cstor
    openebs.io/target-affinity: fio-cstor
```
 
 
#### <a class="anchor" aria-hidden="true" id="volume-tunable"></a>Volume Tunable

Performance tunings based on the workload can be set using Volume Policy. The list of tunings that can be configured are given below:
- <b>queueDepth:</b><br>
 This limits the ongoing IO count from iscsi client on Node to cStor target pod. The default value for this parameter is set at 32.
- <b>luworkers</b> <br>cStor target IO worker threads, sets the number of threads that are working on QueueDepth queue. The default value for this parameter is set at 6. In case of better number of cores and RAM, this value can be 16, which means 16 threads will be running for each volume.
- <b>zvolWorkers:</b><br>cStor volume replica IO worker threads, defaults to the number of cores on the machine. In case of better number of cores and RAM, this value can be 16.

Given below is a sample YAML that has the above parameters configured.

```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
  name: csi-volume-policy
  namespace: openebs
spec:
  replica:
    zvolWorkers: "4"
  target:
    luWorkers: 6
    queueDepth: "32"
```
**Note:**These Policy tunable configurations can be changed for already provisioned volumes by editing the corresponding volume CStorVolumeConfig resources.
 
#### <a class="anchor" aria-hidden="true" id="memory-and-cpu-qos"></a>Memory and CPU Resources QoS
 CStorVolumePolicy can also be used to configure the volume Target pod resource requests and limits to ensure QoS. Given below is a sample YAML that configures the target container's resource requests and limits, and auxResources configuration for the sidecar containers.

<i>To know more about Resource configuration in Kubernetes, <a href="https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/" target="_blank">click here.</a></i>

```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
  name: csi-volume-policy
  namespace: openebs
spec:
  target:
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    auxResources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
```
**Note:** These resource configuration(s) can be changed, for provisioned volumes, by editing the CStorVolumeConfig resource on per volume level.

An example to patch an already existing <code>CStorVolumeConfig</code> resource is given below,
Create a file, say  patch-resources-cvc.yaml, that contains the changes and apply the patch on the resource.
```
spec:
  policy:
    target:
      resources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
          cpu: 250m
          memory: 64Mi
      auxResources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
          cpu: 250m
          memory: 64Mi
```
To apply the patch,
```
kubectl patch cvc -n openebs -p "$(cat patch-resources-cvc.yaml)" pvc-0478b13d-b1ef-4cff-813e-8d2d13bcb316 --type merge
```
#### <a class="anchor" aria-hidden="true" id="#toleration-for-target-pod"></a>Toleration for target pod to ensure scheduling of target pods on tainted nodes

This Kubernetes feature allows users to taint the node. This ensures no pods are be scheduled to it, unless a pod explicitly tolerates the taint. This Kubernetes feature can be used to reserve nodes for specific pods by adding labels to the desired node(s).

One such scenario where the above tunable can be used is: all the volume specific pods, to operate flawlessly, have to be scheduled on nodes that are reserved for storage.

Sample YAML:

```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
  name: csi-volume-policy
  namespace: openebs
spec:
  replica: {}
  target:
    tolerations:
    - key: "key1"
      operator: "Equal"
      value: "value1"
      effect: "NoSchedule"
```
 
#### <a class="anchor" aria-hidden="true" id="priority-class-for-volume-target-deployment"></a> Priority class for volume target deployment
Priority classes can help in controlling the Kubernetes schedulers decisions to favor higher priority pods over lower priority pods. The Kubernetes scheduler can even preempt lower priority pods that are running, so that pending higher priority pods can be scheduled. Setting pod priority also prevents lower priority workloads from impacting critical workloads in the cluster, especially in cases where the cluster starts to reach its resource capacity.
<i>To know more about PriorityClasses in Kubernetes, <a href="https://kubernetes.io/docs/concepts/configuration/pod-priority-preemption/#priorityclass" target="_blank">click here.</a></i>

**Note:** Priority class needs to be created before volume provisioning.

Given below is a sample CStorVolumePolicy YAML which utilises priority class.
```
apiVersion: cstor.openebs.io/v1
kind: CStorVolumePolicy
metadata:
  name: csi-volume-policy
  namespace: openebs
spec:
  provision:
    replicaAffinity: true
  target:
    priorityClassName: "storage-critical"
``` 
### <a class="anchor" aria-hidden="true" id="cstor-cleanup"></a> Cleaning up a cStor setup


Follow the steps below to cleanup of a cStor setup. On successful cleanup you can reuse the cluster's disks/block devices for other storage engines.
1. Delete the application or deployment which uses CSI based cStor CAS engine. In this example we are going to delete the Busybox application that was deployed previously. To delete, execute:
   ```
   kubectl delete pod <pod-name>
   ```

   Example command:
   ```
   kubectl delete busybox
   ```
   Verify that the application pod has been deleted

   ```
   kubectl get pods
   ```
   Sample Output:
   ```
   No resources found in default namespace.
   ```
2. Next, delete the corresponding PVC attached to the application. To delete PVC, execute:
    ```
    kubectl delete pvc <pvc-name>
    ```
    Example command:
    ```
    kubectl delete pvc cstor-pvc
    ```
    Verify that the application-PVC has been deleted.
     ```
     kubectl get pvc
     ```
     Sample Output:
     ```
     No resources found in default namespace.
     ```
3. Delete the corresponding StorageClass used by the application PVC. 
     ```
     kubectl delete sc <storage-class-name>
     ```
     Example command:
     ```
     kubectl delete sc cstor-csi-disk
     ```
     To verify that the StorageClass has been deleted, execute:
     ```
     kubectl get sc
     ```
     Sample Output:
     ```
     No resources found
     ```
4. The blockdevices used to create CSPCs will currently be in claimed state. To get the blockdevice details, execute:
     ```
     kubectl get bd -n openebs
     ```
     Sample Output:
     ```
     NAME                                          NODENAME         SIZE         CLAIMSTATE  STATUS   AGE
     blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68  worker-node-3    21474836480  Claimed     Active   2m10s
     blockdevice-10ad9f484c299597ed1e126d7b857967  worker-node-1    21474836480  Claimed     Active   2m17s
     blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b  worker-node-2    21474836480  Claimed     Active   2m12s

     ```
     To get these blockdevices to unclaimed state delete the associated CSPC. To delete, execute:
     ```
     kubectl delete cspc <CSPC-name> -n openebs
     ```
     Example command:
     ```
     kubectl delete cspc cstor-disk-pool -n openebs
     ```
     Verify that the CSPC and CSPIs have been deleted.
     ```
     kubectl get cspc -n openebs
     ```
     Sample Output:
     ```
     No resources found in openebs namespace.
     ```

     ```
     kubectl get cspi -n openebs
     ```

     Sample Output:
     ```
     No resources found in openebs namespace.
     ```

     Now, the blockdevices must be unclaimed state. To verify, execute:
     ```
     kubectl get bd -n openebs
     ```
     Sample output:
     ```
     NAME                                          NODENAME         SIZE         CLAIMSTATE   STATUS   AGE
     blockdevice-01afcdbe3a9c9e3b281c7133b2af1b68  worker-node-3    21474836480   Unclaimed   Active   21m10s
     blockdevice-10ad9f484c299597ed1e126d7b857967  worker-node-1    21474836480   Unclaimed   Active   21m17s
     blockdevice-3ec130dc1aa932eb4c5af1db4d73ea1b  worker-node-2    21474836480   Unclaimed   Active   21m12s

     ``` 