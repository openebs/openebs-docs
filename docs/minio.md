---
id: minio
title: OpenEBS for MinIO
sidebar_label: MinIO
---
------

<img src="/docs/assets/o-minio.png" alt="OpenEBS and MinIO" style="width:400px;">

<br>

## Introduction

[MinIO](https://github.com/minio/minio) is a high performance distributed object storage server, designed for large-scale private cloud infrastructure. MinIO is designed in a cloud-native manner to scale sustainably in multi-tenant environments. Orchestration platforms like Kubernetes provide a perfect cloud-native environment to deploy and scale MinIO. 

MinIO can be provisioned with OpenEBS volumes using various OpenEBS storage engines such as [Local PV](https://docs.openebs.io/docs/next/localpv.html), [cStor](https://docs.openebs.io/docs/next/cstor.html), or [Jiva](https://docs.openebs.io/docs/next/jiva.html) based on the application requirement. The MinIO operator offers a seamless way to create and update highly available distributed MinIO clusters. MinIO operator brings native support for MinIO, Graphical Console for Admin and Users, and encryption to Kubernetes. It also offers MinIO tenant creation, management, upgrade, zone addition, and more.

Depending on the performance and high availability requirements of MinIO, you can select any of the storage engine to run MinIO with the following deployment options:
- For optimal performance, deploy MinIO with OpenEBS Local PV. 
- If you would like to use storage layer capabilities like high availability, snapshots, incremental backups and restore and so forth, you can select OpenEBS cStor.

This document provides the instructions to setup MinIO operator using OpenEBS Local PV.


## Deployment model

<a href="/docs/assets/svg/Local-PV-Distributed-device-minio.svg" target="_blank"><img src="/docs/assets/svg/Local-PV-Distributed-device-minio.svg" alt="OpenEBS and MinIO Distributed localpv device" style="width:100%;"></a>

In this tutorial, Local PV volume will be provisioned on the node where the application has scheduled and one of the unclaimed and active blockdevice available on the same node will be used to provision the MinIO Object storage. This blockdevice cannot be used by another application. If there are limited blockdevices attached to some of the nodes, then users can use `nodeSelector` in the application YAML to provision application on a particular node where the available blockdevice is present. 

## Configuration workflow

1. [Install OpenEBS](/docs/next/minio.html#install-openebs)
2. [Select OpenEBS storage engine](/docs/next/minio.html#select-openebs-storage-engine)
3. [Configure OpenEBS Local PV StorageClass](/docs/next/minio.html#configure-openebs-local-pv-storageclass)
4. [Install the MinIO plugin](/docs/next/minio.html#install-the-minio-plugin)
5. [Install the MinIO operator deployment](/docs/next/minio.html#install-the-minio-operator-deployment)
6. [Install the MinIO cluster](/docs/next/minio.html#install-the-minio-cluster)
7. [Access MinIO console](/docs/next/minio.html#access-minio-console)

### Install OpenEBS

If OpenEBS is not installed in your K8s cluster, this can be done from [here](https://docs.openebs.io/docs/next/overview.html). If OpenEBS is already installed, go to the next step.

### Select OpenEBS storage engine

A storage engine is the data plane component of the IO path of a Persistent Volume. In CAS architecture, users can choose different data planes for different application workloads based on a configuration policy. OpenEBS provides different types of storage engines and chooses the right engine that suits your type of application requirements and storage available on your Kubernetes nodes. More information can be read from [here](https://docs.openebs.io/docs/next/overview.html#openebs-storage-engines).

In this document, it is mentioned about the installation of MinIO operator using OpenEBS Local PV device. 

### Configure OpenEBS Local PV StorageClass

There are 2 ways to use OpenEBS Local PV.

- `openebs-hostpath` - Using this option, it will create Kubernetes Persistent Volumes that will store the data into OS host path directory at: /var/openebs/<"minio-pv-name">/. Select this option, if you don’t have any additional block devices attached to Kubernetes nodes. You would like to customize the directory where data will be saved, create a new OpenEBS Local PV storage class using these [instructions](https://docs.openebs.io/docs/next/uglocalpv-hostpath.html#create-storageclass). 
  
- `openebs-device` - Using this option, it will create Kubernetes Local PVs using the block devices attached to the node. Select this option when you want to dedicate a complete block device on a node to a MinIO node. You can customize which devices will be discovered and managed by OpenEBS using the instructions [here](https://docs.openebs.io/docs/next/ugndm.html). 

MinIO can provide the replication of data by itself in distributed mode. This method installs MinIO application, which is a StatefulSet kind. It requires a minimum of four (4) nodes to setup MinIO in distributed mode. A distributed MinIO setup with 'n' number of disks/storage has your data safe as long as n/2 or more disks/storage are online. Users should maintain a minimum (n/2 + 1) disks/storage to create new objects. So based on the requirement, the user can choose the appropriate OpenEBS storage engine to run MinIO in distributed mode. For more information on MinIO installation, see MinIO [documentation](https://docs.min.io/docs/deploy-minio-on-kubernetes.html).

The Storage Class `openebs-device` has been chosen to deploy MinIO in the Kubernetes cluster.

### Install the MinIO plugin

The MinIO operator offers MinIO Tenant (MinIO cluster) creation, management of cluster, upgrade, zone addition, and more. Install the MinIO operator plugin using the following command. 
```
$ kubectl krew install minio
```
**Note:** Install `kubectl minio` plugin using krew. Installation of krew can be done from [here](https://krew.sigs.k8s.io/docs/user-guide/setup/install/).

### Install the MinIO operator deployment

Let’s get started by initializing the MinIO operator deployment. This is a one time process.
```
$  kubectl minio init

CustomResourceDefinition tenants.minio.min.io: created
ClusterRole minio-operator-role: created
ServiceAccount minio-operator: created
ClusterRoleBinding minio-operator-binding: created
MinIO Operator Deployment minio-operator: created
``` 
Verify the MinIO operator is successfully installed.
```
$ kubectl get pod

NAME                              READY   STATUS    RESTARTS   AGE
minio-operator-59b8965ff5-tzx8n   1/1     Running   0          18s
```

### Install the MinIO cluster

A tenant is a MinIO cluster created and managed by the operator. Before creating a tenant, please ensure you have requisite nodes and drives in place.
In this guide, we are using 4 Nodes with one 100Gi block device attached per each node. Using the MinIO operator, the following command will generate a YAML file as per the given requirement and the file can be modified as per user specific requirements. 
```
$ kubectl minio tenant create --name tenant1 --servers 4 --volumes 4 --capacity 400Gi -o > tenant.yaml
```
The above will create a YAML spec with 4 MinIO nodes with 100Gi volume. In this YAML file, we need to add the `openebs-device` storage class to create the 100Gi persistent volume using the device attached to each node.

**Note:** Ensure that the image version used for the MinIO console is 0.4.6 or higher. Otherwise, pods will be in `crashloopbackoff` state.

Add the following two changes to the tenant file created using the above command.

- Add the following  to spec.zones.volumeClaimTemplate.spec under Tenant kind.
  ```
  storageClassName: openebs-device
  ```
  An example snippet of the modified tenant YAML file.
  ```
  serviceName: tenant1-internal-service
  zones:
  - resources: {}
    servers: 4
    volumeClaimTemplate:
      apiVersion: v1
      kind: persistentvolumeclaims
      metadata:
        creationTimestamp: null
      spec:
        accessModes:
        - ReadWriteOnce
        storageClassName: openebs-device
        resources:
          requests:
            storage: 100Gi
      status: {}
    volumesPerServer: 1
  ```
- Also, set `requestAutoCert: false` so that MinIO will run in http mode. In this document, we have used http communication for accessing MinIO. The following is a sample snippet of the modified section.
  ```
   mountPath: /export
  requestAutoCert: false
  serviceName: tenant1-internal-service
  ```
Apply the modified tenant YAML spec. The following command will install MinIO tenants under the default namespace.
```
$ kubectl apply -f tenant.yaml

tenant.minio.min.io/tenant1 created
secret/tenant1-creds-secret created
secret/tenant1-console-secret created
```
Verify the MinIO cluster creation is successfully running under the default namespace.
```
$ kubectl get pod

NAME                               READY   STATUS    RESTARTS   AGE
minio-operator-59b8965ff5-tzx8n    1/1     Running   0          6m46s
tenant1-console-6589f7574d-6node   1/1     Running   0          19s
tenant1-console-6589f7574d-wt47v   1/1     Running   0          19s
tenant1-zone-0-0                   1/1     Running   0          51s
tenant1-zone-0-1                   1/1     Running   0          51s
tenant1-zone-0-2                   1/1     Running   0          51s
tenant1-zone-0-3                   1/1     Running   0          50s
```

Verify the MinIO persistent volume details.
```
$ kubectl get pvc

NAME                 STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
0-tenant1-zone-0-0   Bound    pvc-eff2ebdc-1658-4525-b7e2-5d57b39f144b   100Gi      RWO            openebs-device   53s
0-tenant1-zone-0-1   Bound    pvc-1a5881ae-c65a-4ebe-9233-615c6fb7f364   100Gi      RWO            openebs-device   53s
0-tenant1-zone-0-2   Bound    pvc-bd8d3521-fea9-4a48-8f66-26c6d2808997   100Gi      RWO            openebs-device   53s
0-tenant1-zone-0-3   Bound    pvc-55d6aa94-37ed-4f14-bafb-dcee1d7af9f5   100Gi      RWO            openebs-device   52s

$ kubectl get pv

NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                        STORAGECLASS     REASON   AGE
pvc-1a5881ae-c65a-4ebe-9233-615c6fb7f364   100Gi      RWO            Delete           Bound    default/0-tenant1-zone-0-1   openebs-device            49s
pvc-55d6aa94-37ed-4f14-bafb-dcee1d7af9f5   100Gi      RWO            Delete           Bound    default/0-tenant1-zone-0-3   openebs-device            49s
pvc-bd8d3521-fea9-4a48-8f66-26c6d2808997   100Gi      RWO            Delete           Bound    default/0-tenant1-zone-0-2   openebs-device            53s
pvc-eff2ebdc-1658-4525-b7e2-5d57b39f144b   100Gi      RWO            Delete           Bound    default/0-tenant1-zone-0-0   openebs-device            54s
```

Verify MinIO service status.
```
$ kubectl get svc

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)             AGE
kubernetes        ClusterIP   10.100.0.1      <none>        443/TCP             62m
minio             ClusterIP   10.100.59.34    <none>        80/TCP              57s
tenant1-console   ClusterIP   10.100.50.135   <none>        9090/TCP,9443/TCP   25s
tenant1-hl        ClusterIP   None            <none>        9000/TCP            57s
```

Now, MinIO has been installed successfully on your cluster.

**Note:** If the user needs to access MinIO outside the network, the service type can be changed or a new service should be added to use `LoadBalancer` or create `Ingress` resources for production deployment.
For ease of simplicity in testing the deployment, we are going to use `NodePort`. Please be advised to consider using LoadBalancer or Ingress, instead of NodePort, for  production deployment. 

The `minio` service will allow the user to access the console, and `tenant1-console` will allow access to the Admin console. In this guide, we have changed the service type of the services mentioned above, and the following is the output after the modification.

```
$ kubectl get svc

NAME              TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                         AGE
kubernetes        ClusterIP   10.100.0.1      <none>        443/TCP                         64m
minio             NodePort    10.100.59.34    <none>        80:32095/TCP                    3m10s
tenant1-console   NodePort    10.100.50.135   <none>        9090:30383/TCP,9443:30194/TCP   2m38s
tenant1-hl        ClusterIP   None            <none>        9000/TCP                        3m10s
```

### Access MinIO console

There are 2 different console for User and Admin.

#### Access MinIO Admin console

An Admin can access MinIO and do the configuration changes such as creating an account, group, bucket, and its configuration, the setting of user-level permission, file-level permission, etc.

For Admin access, use  `<Node_External_Ip>:<NodePort_of_tenant1-console_service>` in your web browser.

Get the details of Node.
```
$ kubectl get node -o wide

NAME                                            STATUS   ROLES    AGE   VERSION    INTERNAL-IP      EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-192-168-2-55.ap-south-1.compute.internal     Ready    <none>   51m   v1.17.11   192.168.2.55     65.0.121.83      Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-56-236.ap-south-1.compute.internal   Ready    <none>   57m   v1.17.11   192.168.56.236   15.206.189.106   Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-69-83.ap-south-1.compute.internal    Ready    <none>   57m   v1.17.11   192.168.69.83    3.6.91.169       Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-8-117.ap-south-1.compute.internal    Ready    <none>   57m   v1.17.11   192.168.8.117    13.235.210.41    Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
```

Now, access the MinIO service over the browser using the following way.
```
http://3.6.91.169:30383
```
**Note:** Ensure Inbound Rules under VPC-> Security Groups are correctly configured to allow the traffic.

You should enter the Access key and Secret key to login into the admin console. These credentials can be obtained from the secret.
```
$ kubectl get secret tenant1-console-secret -o yaml
```
The following is a sample snippet of the output of the above command. It will show the Access key and Secret key in encoded form. The decoded value should be given in the web browser to login to the user console.
```
apiVersion: v1
data:
  CONSOLE_ACCESS_KEY: MmRkYzA2NGItYTMwZS00ZDg5LTgwODItNWMwYzFkYTRlOGNh
  CONSOLE_HMAC_JWT_SECRET: ODkwYWFlYmEtMTAxYy00YTJmLTg0NDMtYmI1ZjAyMjcyNWFk
  CONSOLE_PBKDF_PASSPHRASE: MDZhN2UzMmUtOWIxZi00MjI2LTk2MmItOTk4OTRmMGYwYjk2
  CONSOLE_PBKDF_SALT: OTg0OTM1YjAtNzgyMS00NWI3LWFmM2ItYzczNDZlNmUzYWNm
  CONSOLE_SECRET_KEY: MGQyY2NlZjktOWM0NC00N2JjLWFkMTYtM2RlNGExMjcwMzY1
kind: Secret
metadata:
  annotations:
```
Decoding of the above credentials can be retrieved by following way.

Access key
```
$ echo 'MmRkYzA2NGItYTMwZS00ZDg5LTgwODItNWMwYzFkYTRlOGNh' | base64 -d
2ddc064b-a30e-4d89-8082-5c0c1da4e8ca
```

Secret key
```
$ echo 'MGQyY2NlZjktOWM0NC00N2JjLWFkMTYtM2RlNGExMjcwMzY1' | base64 -d
0d2ccef9-9c44-47bc-ad16-3de4a1270365
```

#### Access MinIO User console

The MinIO StatefulSet application is created using NodePort as the service type. To access MinIO over a web browser, use `<Node_External_Ip>:<NodePort_of_minio_service>` this way. 

Get the details of Node.
```
$ kubectl get node -o wide

NAME                                            STATUS   ROLES    AGE   VERSION    INTERNAL-IP      EXTERNAL-IP      OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
ip-192-168-2-55.ap-south-1.compute.internal     Ready    <none>   51m   v1.17.11   192.168.2.55     65.0.121.83      Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-56-236.ap-south-1.compute.internal   Ready    <none>   57m   v1.17.11   192.168.56.236   15.206.189.106   Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-69-83.ap-south-1.compute.internal    Ready    <none>   57m   v1.17.11   192.168.69.83    3.6.91.169       Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
ip-192-168-8-117.ap-south-1.compute.internal    Ready    <none>   57m   v1.17.11   192.168.8.117    13.235.210.41    Ubuntu 18.04.5 LTS   5.4.0-1028-aws   docker://17.3.2
```

Now, access the MinIO service over the browser using the following way.
```
http://3.6.91.169:32095
```

You should enter the `Access key` and `Secret key` to login into the user console. These credentials can be obtained from the secret.
```
$ kubectl get secret tenant1-creds-secret -o yaml
```
The following is a sample snippet of the output of the above command. It will show the Access key and Secret key in encoded form. The decoded value should be given in the web browser to login to the user console.
```
apiVersion: v1
data:
  accesskey: N2MyMTI0MWItZDczOS00NDEwLWE0OWQtOTkyODkwNDNiNDQ1
  secretkey: M2ZiNGFlZGQtYTU1Yy00YjM4LWJkNTQtODEyNmViOTg5ZmZk
kind: Secret
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"accesskey":"N2MyMTI0MWItZDczOS00NDEwLWE0OWQtOTkyODkwNDNiNDQ1","secretkey":"M2ZiNGFlZGQtYTU1Yy00YjM4LWJkNTQtODEyNmViOTg5ZmZk"},"kind":"Secret","metadata":{"annotations":{},"creationTimestamp":null,"name":"tenant1-creds-secret","namespace":"default"}}
```
Decoding of the above credentials can be retrieved by following way.

Access key
```
$ echo 'N2MyMTI0MWItZDczOS00NDEwLWE0OWQtOTkyODkwNDNiNDQ1' | base64 -d
7c21241b-d739-4410-a49d-99289043b445
```
Secret key
```
$ echo 'M2ZiNGFlZGQtYTU1Yy00YjM4LWJkNTQtODEyNmViOTg5ZmZk' | base64 -d
3fb4aedd-a55c-4b38-bd54-8126eb989ffd
```

<br>

<hr>

## See Also:

### [OpenEBS use cases](/docs/next/usecases.html)

### [Understanding NDM](/docs/next/ugndm.html)

### [Local PV concepts](/docs/next/localpv.html)

### [Local PV User guide](/docs/next/uglocalpv-device.html)

<hr>

<br>





<!-- Hotjar Tracking Code for https://docs.openebs.io -->

<script>
   (function(h,o,t,j,a,r){
       h.hj=h.hj||function(){(h.hj.q=h.hj.q||[]).push(arguments)};
       h._hjSettings={hjid:785693,hjsv:6};
       a=o.getElementsByTagName('head')[0];
       r=o.createElement('script');r.async=1;
       r.src=t+h._hjSettings.hjid+j+h._hjSettings.hjsv;
       a.appendChild(r);
   })(window,document,'https://static.hotjar.com/c/hotjar-','.js?sv=');
</script>


<!-- Global site tag (gtag.js) - Google Analytics -->
<script async src="https://www.googletagmanager.com/gtag/js?id=UA-92076314-12"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'UA-92076314-12');
</script>
