---
id: mongo
title: OpenEBS for MongoDB
sidebar_label: MongoDB
---

<img src="/docs/assets/svg/o-mongo.svg" alt="OpenEBS and MongoDB" style="width:400px;">

## Introduction

<br>

MongoDB is a cross-platform document-oriented database. Classified as a NoSQL database, MongoDB eschews the traditional table-based relational database structure in favor of JSON-like documents with dynamic schemas, making the integration of data in certain types of applications easier and faster. MongoDB  is deployed usually as a `Statefulset` on Kubernetes and requires persistent storage for each instance of MongoDB Storage Manager instance. OpenEBS provides persistent volumes on the fly when Storage Managers are scaled up.

<br>

**Advantages of using OpenEBS for MongoDB:**

- No need to manage the local disks, they are managed by OpenEBS
- Large size PVs can be provisioned by OpenEBS and MongoDB
- Start with small storage and add disks as needed on the fly. Sometimes MongoDB instances are scaled up because of capacity on the nodes. With OpenEBS persistent volumes, capacity can be thin provisioned and disks can be added to OpenEBS on the fly without disruption of service 
- MongoDB sometimes need highly available storage, in such cases OpenEBS volumes can be configured with 3 replicas.
- If required, take backup of the MongoDB data periodically and back them up to S3 or any object storage so that restoration of the same data is possible to the same or any other Kubernetes cluster

<br>

*Note: MongoDB can be deployed both as `Deployment` or as `StatefulSet`. When MongoDB deployed as `StatefulSet`, you don't need to replicate the data again at OpenEBS level. When MongoDB is deployed as `Deployment`, consider 3 OpenEBS replicas, choose the StorageClass accordingly.*

<br>

<hr>

<br>

## Deployment model

<br>

<a href="/docs/assets/svg/mongo-deployment.svg" target="_blank"><img src="/docs/assets/svg/mongo-deployment.svg" alt="OpenEBS and Mongo" style="width:100%;"></a>

<br>

<hr>

<br>

## Configuration workflow

<br>

1. **Install OpenEBS**

   If OpenEBS is not installed in your K8s cluster, this can done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step. 

2. **Configure cStor Pool**

   After OpenEBS installation, cStor pool has to be configured. If cStor Pool is not configured in your OpenEBS cluster, this can be done from [here](/docs/next/ugcstor.html#creating-cStor-storage-pools).  During cStor Pool creation, make sure that the maxPools parameter is set to >=3. Sample YAML named **openebs-config.yaml** for configuring cStor Pool is provided in the Configuration details below. If cStor pool is already configured, go to the next step. 

4. **Create Storage Class**

   You must configure a StorageClass to provision cStor volume on given cStor pool. StorageClass is the interface through which most of the OpenEBS storage policies are defined. In this solution we are using a StorageClass to consume the cStor Pool which is created using external disks attached on the Nodes.  In this solution, MongoDB is installed as a Deployment. So it requires replication at the storage level. So cStor volume `replicaCount` is 3. Sample YAML named **openebs-sc-disk.yaml** to consume cStor pool with cStove volume replica count as 3 is provided in the configuration details below.

5. **Launch and test MongoDB**

   Use stable MongoDB chart with helm to install MongoDB deployment in your cluster using the following command. In the following command, it will create a PVC with 8Gi size with a storage replication factor 3.

   ```
   helm install --name my-release --set persistence.storageClass=openebs-cstor-disk stable/mongodb
   ```

   For more information on installation, see MongoDB [documentation](https://github.com/helm/charts/tree/master/stable/mongodb).



## Reference at [openebs.ci](https://openebs.ci/)

<br>

A live deployment of MongoDB using OpenEBS volumes can be seen at the website [www.openebs.ci](https://openebs.ci/)

Deployment YAML spec files for MongoDB and OpenEBS resources are found [here](https://github.com/openebs/e2e-infrastructure/blob/54fe55c5da8b46503e207fe0bc08f9624b31e24c/production/mongo-cstor/mongo-cstor-mongo.yaml)

[OpenEBS-CI dashboard of MongoDB](https://openebs.ci/mongo-cstor)



<br>

<hr>

<br>



## Post deployment Operations

<br>

**Monitor OpenEBS Volume size** 

It is not seamless to increase the cStor volume size (refer to the roadmap item). Hence, it is recommended that sufficient size is allocated during the initial configuration.

**Monitor cStor Pool size**

As in most cases, cStor pool may not be dedicated to just Mongo database alone. It is recommended to watch the pool capacity and add more disks to the pool before it hits 80% threshold. See [cStorPool metrics](/docs/next/ugcstor.html#monitor-pool).



<br>

<hr>

<br>





## Configuration details

<br>

**openebs-config.yaml**

```
#Use the following YAMLs to create a cStor Storage Pool.
# and associated storage class.
apiVersion: openebs.io/v1alpha1
kind: StoragePoolClaim
metadata:
  name: cstor-disk
spec:
  name: cstor-disk
  type: disk
  poolSpec:
    poolType: striped
  # NOTE - Appropriate disks need to be fetched using `kubectl get blockdevices -n openebs`
  # `Block devices` is a custom resource supported by OpenEBS with `node-disk-manager`
  # as the disk operator
# Replace the following with actual disk CRs from your cluster `kubectl get blockdevices -n openebs`
# Uncomment the below lines after updating the actual disk names.
  blockDevices:
    blockDeviceList:
# Replace the following with actual disk CRs from your cluster from `kubectl get blockdevices -n openebs`
#   - blockdevice-69cdfd958dcce3025ed1ff02b936d9b4
#   - blockdevice-891ad1b581591ae6b54a36b5526550a2
#   - blockdevice-ceaab442d802ca6aae20c36d20859a0b
---
```

**openebs-sc-disk.yaml**

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: openebs-cstor-disk
  annotations:
    openebs.io/cas-type: cstor
    cas.openebs.io/config: |
      - name: StoragePoolClaim
        value: "cstor-disk"
      - name: ReplicaCount
        value: "3"       
provisioner: openebs.io/provisioner-iscsi
reclaimPolicy: Delete
---
```



<br>

## See Also:

<br>

### [OpenEBS architecture](/docs/next/architecture.html)

### [OpenEBS use cases](/docs/next/usecases.html)

### [cStor pools overview](/docs/next/cstor.html#cstor-pools)



<br>

<hr>
<br>
