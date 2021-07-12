---
id: gitlab
title: OpenEBS for GitLab
sidebar_label: GitLab
---
------

<img src="/docs/assets/o-gitlab.png" alt="OpenEBS and GitLab" style="width:400px;">





## Introduction

<br>

GitLab is a good solution for building On-Premise cloud native CI/CD platforms, it is a single application for the entire software development lifecycle. The helm charts for GitLab are made so simple that the entire infrastructure including the underlying databases and storage needed for GitLab are dynamically provisioned. This solution discusses the use cases of using OpenEBS from a single pool of storage for all the databases required to run GitLab.



**Advantages of using OpenEBS for Gitlab:**

- OpenEBS acts a single storage platform for all stateful applications including Gitaly,  Redis, PostgreSQL, Minio and Prometheus

- OpenEBS volumes are highly available. Node loss, reboots and Kubernetes upgrades will not affect the availability of persistent storage to the stateful applications

- Storage is scalable on demand. You can start with a small storage for all the databases required by GitLab and scale it on demand

  

<br>

<hr>

<br>

## Deployment model

<br>



<a href="/docs/assets/svg/gitlab-deployment.svg" target=""><img src="/docs/assets/svg/gitlab-deployment.svg" alt="GitLab deployment using OpenEBS" style="width:100%;"></a>

<br>

<br>

<hr>

<br>

## Configuration workflow

<br>

1. **Install OpenEBS**

   If OpenEBS is not installed in your K8s cluster, this can done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step. 

2. **Configure cStor Pool**

   After OpenEBS installation, cStor pool has to be configured. If cStor Pool is not configured in your OpenEBS cluster, this can be done from [here](/docs/next/ugcstor.html#creating-cStor-storage-pools). During cStor Pool creation, make sure that the maxPools parameter is set to >=3. Sample YAML named **openebs-config.yaml** for configuring cStor Pool is provided in the Configuration details below. If cStor pool is already configured, go to the next step. 

4. **Create Storage Class**

   You must configure a StorageClass to provision cStor volume on given cStor pool. StorageClass is the interface through which most of the OpenEBS storage policies are defined. In this solution we are using a StorageClass to consume the cStor Pool which is created using external disks attached on the Nodes.  Since GitLab is a StatefulSet application and it requires only single storage replication. So cStor volume `replicaCount` is =1. Sample YAML named **openebs-sc-disk.yaml**to consume cStor pool with cStor volume replica count as 1 is provided in the configuration details below.

5. **Launch and test GitLab**

   Patch your StorageClass which is going to be used for the GitLab installation using the following command. 

   ```
   kubectl patch storageclass openebs-cstor-disk -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
   ```

   Use stable Gitlab image with helm to deploy GitLab in your cluster using the following command. In the following command, it will create two PVCs such as 1Gi for storing generated configuration files, keys, and certs and 10Gi is used to store git data and other project files.

   ```
   helm repo add gitlab https://charts.gitlab.io/
   helm repo update
   helm upgrade --install gitlab gitlab/gitlab \
     --timeout 600 \
     --set global.hosts.domain=<domain_name>\
     --set global.hosts.externalIP=<GitLab_Service_IP> \
   ```

   For more information on installation, see GitLab [documentation](https://docs.gitlab.com/ee/install/kubernetes/gitlab_chart.html#deployment-of-gitlab-to-kubernetes).

   **Note:** You may be required to add "fsGroup:1000" under "spec.template.spec.securityContext" in corresponding gitlab-prometheus-server deployment spec for writing metrics to it.

<br>

<hr>

<br>

## Post deployment Operations

<br>

**Monitor OpenEBS Volume size** 

It is not seamless to increase the cStor volume size (refer to the roadmap item). Hence, it is recommended that sufficient size is allocated during the initial configuration.

**Monitor cStor Pool size**

As in most cases, cStor pool may not be dedicated to just GitLab's databases alone. It is recommended to watch the pool capacity and add more disks to the pool before it hits 80% threshold. See [cStorPool metrics](/docs/next/ugcstor.html#monitor-pool). 

**Maintain volume replica quorum during node upgrades**

 cStor volume replicas need to be in quorum when applications are deployed as `deployment` and cStor volume is configured to have `3 replicas`. Node reboots may be common during Kubernetes upgrade. Maintain volume replica quorum in such instances. See [here](/docs/next/k8supgrades.html) for more details.

<br>

<hr>

<br>

## Configuration details

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
  # NOTE - Appropriate disks need to be fetched using `kubectl get blockdevices -n openebs`
  #
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
        value: "1"       
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
