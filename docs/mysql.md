---
id: mysql
title: Managed database service like RDS
sidebar_label: RDS like MySQL
---
------

<img src="/docs/assets/o-mysql.png" alt="OpenEBS and MySQL" style="width:400px;">

## Introduction

<br>

 MYSQL is most ubiquitous and commonly used database. RDS is a service that make deploying and making MYSQL easy. With OpenEBS CAS architecture each user is assigned an independent stack of storage that serves just one instance of MySQL database, making it easy to handle the provisioning and post deployment operations like capacity increase and upgrades. 



Use OpenEBS and MySQL containers to quickly launch an RDS like service, where database launch is instantaneous, availability is provided across zones and increase in requested capacity can happen on the fly. 

<br>

<hr>

<br>

## Deployment model 

<br>

<a href="/docs/assets/svg/mysql-deployment.svg" target="_blank"><img src="/docs/assets/svg/mysql-deployment.svg" alt="OpenEBS and Percona" style="width:100%;"></a>



As shown above, OpenEBS volumes need to be configured with three replicas for high availability. This configuration work fine when the nodes (hence the cStor pool) is deployed across Kubernetes zones.

<br>

<hr>

<br>

## Configuration workflow

<br>

1. **Install OpenEBS**

   If OpenEBS is not installed in your K8s cluster, this can done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step. 

2. **Configure cStor Pool** : After OpenEBS installation,  cStor pool has to be configured. As MySQL is a deployment, it need high availability at storage level. OpenEBS cStor volume has to be configured with 3 replica. During cStor Pool creation, make sure that the maxPools parameter is set to >=3. If cStor Pool is already configured as required go to Step 4 to create MySQL StorageClass. 

4. **Create Storage Class**

   You must configure a StorageClass to provision cStor volume on cStor pool. StorageClass is the interface through which most of the OpenEBS storage policies  are defined. In this solution we are using a StorageClass to consume the cStor Pool which is created using external disks attached on the Nodes. Since MySQL is a deployments, it requires high availability of data at storage level. So cStor volume `replicaCount` is 3. Sample YAML named **openebs-sc-disk.yaml**to consume cStor pool with cStor volume replica count as 3 is provided in the configuration details below.

5. **Launch and test Mysql**:

   Run MySQL database application using the stable helm chart. The following command  will install MySQL database application in your cluster. This command will create a PVC of size 8Gi for running the database.

   ```
   helm install --name my-release --set persistence.storageClass=openebs-cstor-disk stable/mysql
   ```

    For more information about installation, see MySQL [documentation](https://github.com/helm/charts/tree/master/stable/mysql).

<br>

<hr>

<br>

## Post deployment Operations

<br>

**Monitor OpenEBS Volume size** 

It is not seamless to increase the cStor volume size (refer to the roadmap item). Hence, it is recommended that sufficient size is allocated during the initial configuration. 

**Monitor cStor Pool size**

As in most cases, cStor pool may not be dedicated to just MySQL alone. It is recommended to watch the pool capacity and add more disks to the pool before it hits 80% threshold. See [cStorPool metrics](/docs/next/ugcstor.html#monitor-pool) 

**Maintain volume replica quorum during node upgrades**

 cStor volume replicas need to be in quorum when MySQL application is deployed as `deployment` and cStor volume is configured to have `3 replicas`. Node reboots may be common during Kubernetes upgrade. Maintain volume replica quorum in such instances. See [here](/docs/next/k8supgrades.html) for more details.

<br>

<hr>

<br>

## Configuration Details

<br>

**openebs-config.yaml**

```yaml

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
  # NOTE - Appropriate disks need to be fetched using `kubectl get bd -n openebs`
  #
  # `Block Devices` is a custom resource supported by OpenEBS with `node-disk-manager`
  # as the disk operator
# Replace the following with actual disk CRs from your cluster `kubectl get bd  -n openebs`
# Uncomment the below lines after updating the actual disk names.
  blockdevices:
    blockDeviceList:
# Replace the following with actual disk CRs from your cluster from `kubectl get bd  -n openebs`
#   - blockdevice-69cdfd958dcce3025ed1ff02b936d9b4
#   - blockdevice-891ad1b581591ae6b54a36b5526550a2
#   - blockdevice-ceaab442d802ca6aae20c36d20859a0b
  
---
```

**openebs-sc-disk.yaml**

```yaml

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

<hr>

<br>

## See Also:

<br>

### [OpenEBS architecture](/docs/next/architecture.html)

### [OpenEBS use cases](/docs/next/usecases.html)

### [cStor pools overview](/docs/next/cstor.html#cstor-pools)



<br>

<hr>

<br>

