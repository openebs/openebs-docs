---
id: percona
title: OpenEBS for Percona
sidebar_label: Percona
---
------

<img src="/docs/assets/o-percona.png" alt="OpenEBS and Percona" style="width:400px;">


## Introduction

The Percona XtraDB Cluster (PXC) is a fully open-source high-availability solution for MySQL. It integrates Percona Server for MySQL and Percona XtraBackup with the Galera library to enable synchronous multi-master replication. A cluster consists of nodes, where each node contains the same set of data synchronized across nodes. The recommended configuration is to have at least three nodes. Each node is a regular Percona Server for MySQL instances. 

Percona XtraDB Cluster can be provisioned with OpenEBS volumes using OpenEBS storage engine- OpenEBS Local PV. Depending on the performance and high availability requirements of Percona, you can select any of the storage engine to run Percona with the following deployment options:
- For optimal performance, deploy Percona PXC with OpenEBS Local PV. 
- If you would like to use storage layer capabilities like high availability, snapshots, incremental backups and restore and so forth, you can select OpenEBS cStor. 

<br>

## Deployment model 

<br>

<a href="/docs/assets/svg/percona-deployment-new.svg" target="_blank"><img src="/docs/assets/svg/percona-deployment-new.svg" alt="OpenEBS and Percona" style="width:100%;"></a>

This tutorial provides detailed instructions to run a Percona XtraDB Cluster(PXC) with OpenEBS Local PV and perform some simple database operations to verify the successful deployment and it's performance benchmark.

<br>

## Configuration workflow 

1. Install OpenEBS
2. Select OpenEBS storage engine
3. Configure OpenEBS Local PV StorageClass
4. Install the Percona XtraDB Cluster operator
5. Update Storage and Monitoring section
6. Install the Percona XtraDB Cluster
7. Access Percona MySQL database
8. Run performance benchmark

<br>

### Install OpenEBS

If OpenEBS is not installed in your K8s cluster, this can be done from [here](https://docs.openebs.io/docs/next/overview.html). If OpenEBS is already installed, go to the next step.

### Select OpenEBS storage engine

A storage engine is the data plane component of the IO path of a Persistent Volume. In CAS architecture, users can choose different data planes for different application workloads based on a configuration policy. OpenEBS provides different types of storage engines. Choose the right engine that suits your type of application requirements and storage available on your Kubernetes nodes. More information can be read from [here](https://docs.openebs.io/docs/next/overview.html#openebs-storage-engines).

### Configure OpenEBS Local PV StorageClass

In this tutorial, OpenEBS Local PV device has been used as the storage engine for deploying Percona PXC. There are 2 ways to use OpenEBS Local PV.

- `openebs-hostpath` - Using this option, it will create Kubernetes Persistent Volumes that will store the data into OS host path directory at: /var/openebs/<percona-pv>/. Select this option, if you don’t have any additional block devices attached to Kubernetes nodes. You would like to customize the directory where data will be saved, create a new OpenEBS Local PV storage class using these [instructions](https://docs.openebs.io/docs/next/uglocalpv-hostpath.html#create-storageclass). 
  
- `openebs-device` - Using this option, it will create Kubernetes Local PVs using the block devices attached to the node. Select this option when you want to dedicate a complete block device on a node to a Percona node. You can customize which devices will be discovered and managed by OpenEBS using the instructions [here](https://docs.openebs.io/docs/next/ugndm.html). 

The Storage Class `openebs-device` has been chosen to deploy PXC in the Kubernetes cluster.

### Install the Percona XtraDB Cluster operator
```
$ git clone -b v1.6.0 https://github.com/percona/percona-xtradb-cluster-operator
$ cd percona-xtradb-cluster-operator
$ kubectl apply -f deploy/bundle.yaml
```
Verify if the operator is running correctly
```
$ kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
percona-xtradb-cluster-operator-749b86b678-8f4q5   1/1     Running   0          23s
```

#### Update Storage and Monitoring specification

In this document, we have made changes in the storage section for PXC and the monitoring section PMM.

##### Changes done in the Storage section for PXC: 

Update Storage Class name and required storage parameters in deploy/cr.yaml. In this example, we have updated the following parameters: 
```
spec.pxc.volumeSpec.persistentVolumeClaim.storageClassName as “openebs-device” and 
spec.pxc.volumeSpec.persistentVolumeClaim.resources.requests.storage as “100Gi”
```
Sample snippet:
```
    volumeSpec:
      persistentVolumeClaim:
        storageClassName: openebs-device
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 100Gi
```
**Note:** Ensure you have 100Gi is attached with each Node. Else, provide the storage capacity as per the capacity of the available disk. 

###### Changes done in the Monitoring section for PMM:

Enable monitoring service and server user name. In this example, we have updated the following parameters:
```
spec.pmm.enabled as “true”
spec.pmm.serverUser as “admin”  
```
The following is the sample snippet of PMM spec of Percona XtraDB where we enabled monitoring feature and updated the PMM server username.
```
  pmm:
    enabled: true
    image: percona/percona-xtradb-cluster-operator:1.6.0-pmm
    serverHost: monitoring-service
    serverUser: admin
```

### Install the Percona XtraDB Cluster

There is a dependency if you are enabling a monitoring service(PMM) for your PXC. In this case, you must install the PMM server using the following command before installing PXC. We have used Percona [blog](https://www.percona.com/blog/2020/07/23/using-percona-kubernetes-operators-with-percona-monitoring-and-management/) to enable the monitoring service.

#### Use Helm to install PMM Server

Using helm, add the Percona chart repository and update the information for the available charts as follows:
```
$ helm repo add percona https://percona-charts.storage.googleapis.com
$ helm repo update
$ helm install monitoring percona/pmm-server --set platform=kubernetes --version 2.7.0 --set "credentials.password=test123" 
```
**Note:** In this document, we have used “test123” as the PMM server credential password and the base64 encoded form of this password is “dGVzdDEyMw==”. This encoded value will be added in one of the secrets while installing the PXC cluster and also while running the performance benchmark task.

Now, verify PMM server pod is installed and running.
```
$ kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
monitoring-0                                       1/1     Running   0          70m
```
In the previous section, we have made the required changes on the CR YAML spec. Let’s install the PXC cluster using the following command. Ensure your current directory is the cloned Percona directory.
```
$ kubectl apply -f deploy/cr.yaml
```
After applying the above command, you may see that cluster1-pxc-0 pod started in `CreateContainerConfigError` state. 
```
$  kubectl get pod
NAME                                               READY   STATUS                       RESTARTS   AGE
cluster1-haproxy-0                                 1/2     Running                      0          21s
cluster1-pxc-0                                     0/2     CreateContainerConfigError   0          21s
monitoring-0                                       1/1     Running                      0          107m
percona-xtradb-cluster-operator-749b86b678-8f4q5   1/1     Running                      0          3h50m
```
This is due to the unavailability of the PMM server key in the secret. To resolve this, edit the corresponding secret and add the PMM server key. 

```
$ kubectl get secrets
NAME                                          TYPE                                  DATA   AGE
default-token-s78cq                           kubernetes.io/service-account-token   3      5h15m
internal-cluster1                             Opaque                                6      65s
my-cluster-secrets                            Opaque                                6      65s
my-cluster-ssl                                kubernetes.io/tls                     3      61s
my-cluster-ssl-internal                       kubernetes.io/tls                     3      60s
percona-xtradb-cluster-operator-token-v82b5   kubernetes.io/service-account-token   3      3h50m
sh.helm.release.v1.monitoring.v1              helm.sh/release.v1                    1      107m
```
Let’s edit the secret `internal-cluster1` using the following command and add `pmmserver` value as per the given credential password during PMM server installation time. In this example, we have added  `pmmserver: dGVzdDEyMw==`  in the secret `internal-cluster1`. 

```
$ kubectl edit secret internal-cluster1
```
Sample spec of the modified secret content.
```
apiVersion: v1
data:
  clustercheck: dUt5QVlMYTVKdWxaZDA1NGI=
  monitor: NkRwbFVJcExCSFFFSHBMM3k=
  operator: OGsyMmxhaG02blh0aW9BbkFW
  proxyadmin: cVB6elZHZXUwVWNkaUV4MTJp
  root: WnV2cFNiRGU4UWhpWjNmd1Y=
  pmmserver: dGVzdDEyMw==
  xtrabackup: MmVGSGsyWTlJdk44ZUlmQXlnYQ==
kind: Secret
```

Now, verify that all required components are installed and running successfully.
```
$ kubectl get pod
NAME                                               READY   STATUS    RESTARTS   AGE
cluster1-haproxy-0                                 2/2     Running   0          4m33s
cluster1-haproxy-1                                 2/2     Running   0          2m40s
cluster1-haproxy-2                                 2/2     Running   0          2m21s
cluster1-pxc-0                                     2/2     Running   0          4m33s
cluster1-pxc-1                                     2/2     Running   0          2m46s
cluster1-pxc-2                                     2/2     Running   0          92s
monitoring-0                                       1/1     Running   0          111m
percona-xtradb-cluster-operator-749b86b678-8f4q5   1/1     Running   0          3h54m

$ kubectl get pvc
NAME                     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
datadir-cluster1-pxc-0   Bound    pvc-3c0ef6d5-6d04-469b-aad1-e4a6881a5176   10Gi       RWO            openebs-device   4m49s
datadir-cluster1-pxc-1   Bound    pvc-92ed74a0-ccf7-48e1-8b5a-721ea87d4282   10Gi       RWO            openebs-device   3m2s
datadir-cluster1-pxc-2   Bound    pvc-1f9f619c-998d-4f60-9ed8-7b785c5cb49e   10Gi       RWO            openebs-device   108s
pmmdata-monitoring-0     Bound    pvc-02514e16-4f3f-4123-96ae-7f609c9377f7   8Gi        RWO            gp2              3h42m

$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                            STORAGECLASS     REASON   AGE
pvc-02514e16-4f3f-4123-96ae-7f609c9377f7   8Gi        RWO            Delete           Bound    default/pmmdata-monitoring-0     gp2                       3h42m
pvc-1f9f619c-998d-4f60-9ed8-7b785c5cb49e   10Gi       RWO            Delete           Bound    default/datadir-cluster1-pxc-2   openebs-device            117s
pvc-3c0ef6d5-6d04-469b-aad1-e4a6881a5176   10Gi       RWO            Delete           Bound    default/datadir-cluster1-pxc-0   openebs-device            4m59s
pvc-92ed74a0-ccf7-48e1-8b5a-721ea87d4282   10Gi       RWO            Delete           Bound    default/datadir-cluster1-pxc-1   openebs-device            3m11s

$ kubectl get svc
NAME                        TYPE           CLUSTER-IP       EXTERNAL-IP                                                                    PORT(S)                       AGE
cluster1-haproxy            ClusterIP      10.100.136.90    <none>                                                                         3306/TCP,3309/TCP,33062/TCP   5m18s
cluster1-haproxy-replicas   ClusterIP      10.100.244.115   <none>                                                                         3306/TCP                      5m18s
cluster1-pxc                ClusterIP      None             <none>                                                                         3306/TCP,33062/TCP            5m18s
cluster1-pxc-unready        ClusterIP      None             <none>                                                                         3306/TCP,33062/TCP            5m18s
kubernetes                  ClusterIP      10.100.0.1       <none>                                                                         443/TCP                       5h19m
monitoring-service          LoadBalancer   10.100.32.246    a543e9e1d189644f9bf4f7fdf0ba15b3-1159960729.ap-southeast-1.elb.amazonaws.com   443:30317/TCP                 112m
```

### Access Percona MySQL database

```
$ kubectl get secret my-cluster-secrets -o yaml
```
Sample snippet of output:

```
apiVersion: v1
data:
  clustercheck: dUt5QVlMYTVKdWxaZDA1NGI=
  monitor: NkRwbFVJcExCSFFFSHBMM3k=
  operator: OGsyMmxhaG02blh0aW9BbkFW
  proxyadmin: cVB6elZHZXUwVWNkaUV4MTJp
  root: WnV2cFNiRGU4UWhpWjNmd1Y=
  xtrabackup: MmVGSGsyWTlJdk44ZUlmQXlnYQ==
kind: Secret
```

Now, get the encoded information of the data named as `root`. It is given as “WnV2cFNiRGU4UWhpWjNmd1Y=”. The decoded value can be found using the following method.

```
$ echo 'WnV2cFNiRGU4UWhpWjNmd1Y=' | base64 -d
ZuvpSbDe8QhiZ3fwV
```

Let’s run a Percona client to perform the database operations. You can run simple database operations in many ways. One method is by logging in to any of the Percona pods and running MySQL commands. In this example, we have created a Percona Client pod, and by using this pod, database operations are performed.

The following command will run a Percona client pod through which we can access the PXC cluster and perform database operations. Once you enter into the Percona client shell, login to the MySQL console by providing the user credentials. In this case, username as “root” and password should be the decoded value, which can be found above.

```
$ kubectl run -i --rm --tty percona-client --image=percona:8.0 --restart=Never -- bash -il

percona-client:/$ mysql -h cluster1-haproxy -uroot -pZuvpSbDe8QhiZ3fwV
mysql: [Warning] Using a password on the command line interface can be insecure.
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 3352
Server version: 8.0.20-11.2 Percona XtraDB Cluster (GPL), Release rel11, Revision 9132e55, WSREP version 26.4.3

Copyright (c) 2009-2020 Percona LLC and/or its affiliates
Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
mysql> 

Let’s do some simple database operations.

mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
+--------------------+
4 rows in set (0.01 sec)

mysql> CREATE DATABASE sbtest;
Query OK, 1 row affected (0.02 sec)

Use this same Database name in the following performance benchmark tasks. If you use any non-existence database name in the performance benchmark, the command will fail.


mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sbtest             |
| sys                |
+--------------------+
5 rows in set (0.00 sec)

mysql> use sbtest;
Database changed


mysql> create table sbtest_book(
    ->    tutorial_id INT NOT NULL AUTO_INCREMENT,
    ->    tutorial_title VARCHAR(100) NOT NULL,
    ->    tutorial_author VARCHAR(40) NOT NULL,
    ->    submission_date DATE,
    ->    PRIMARY KEY ( tutorial_id )
    -> );
Query OK, 0 rows affected (0.03 sec)

$ INSERT INTO sbtest_book (tutorial_title, tutorial_author, submission_date)
VALUES ("Learn Kubera", "MayaData", NOW());
Query OK, 1 row affected, 1 warning (0.02 sec)


mysql> select * from sbtest_book;
+-------------+----------------+-----------------+-----------------+
| tutorial_id | tutorial_title | tutorial_author | submission_date |
+-------------+----------------+-----------------+-----------------+
|           1 | Learn Kubera   | MayaData        | 2020-12-08      |
+-------------+----------------+-----------------+-----------------+

1 row in set (0.00 sec)
mysql> exit
Bye
[mysql@percona-client /]$ exit
logout
```

### Run Percona Performance benchmark 

Let’s create a SysBench pod to perform the performance benchmark of the PXC database.

```
$ kubectl run -it --rm sysbench-client --image=perconalab/sysbench:latest --restart=Never -- bash
If you don't see a command prompt, try pressing Enter.
root@sysbench-client:/sysbench#
```

The above command will create a temporary pod for SysBench. This pod will be used to run the benchmark commands. In this example, we are using the PXC service name as the mysql host in the following performance benchmark test command. The root password used in the following command can be obtained from the previous section. 
Run the following tests from the SysBench pod.
 
#### Prepare the data

Ensure the same database has already been created before running the tests. In this example, we have created a database called “sbtest” in the previous section and used it in the performance benchmark tests. Please remember to use the corresponding MySQL password throughout the performance benchmark tests. 
```
root@sysbench-client:/sysbench# sysbench oltp_read_only --tables=10 --table_size=1000000 --mysql-port=3306 --mysql-db=sbtest --mysql-host=cluster1-pxc --mysql-user=root --mysql-password=ZuvpSbDe8QhiZ3fwV prepare
```

#### Perform Read-only test
```
root@sysbench-client:/sysbench# sysbench oltp_read_only --tables=10 --table_size=1000000 --threads=8 --mysql-port=3306 --mysql-db=sbtest  --mysql-host=cluster1-pxc --mysql-user=root --mysql-password=ZuvpSbDe8QhiZ3fwV --report-interval=1 --time=300 run
```

#### Perform Write-only test
```
root@sysbench-client:/sysbench# sysbench oltp_write_only --tables=10 --table_size=1000000 --threads=8 --mysql-port=3306 --mysql-db=sbtest --mysql-host=cluster1-pxc --mysql-user=root --mysql-password=ZuvpSbDe8QhiZ3fwV --time=60 --report-interval=1 run
```

#### Perform Read-Write test

```
root@sysbench-client:/sysbench# sysbench oltp_read_write --tables=10 --table_size=1000000 --threads=8 --mysql-port=3306 --mysql-db=sbtest --mysql-host=cluster1-pxc --mysql-user=root --mysql-password=ZuvpSbDe8QhiZ3fwV --time=300 --report-interval=1 run
```


## See Also:

<br>

### [OpenEBS architecture](/docs/next/architecture.html)

### [OpenEBS use cases](/docs/next/usecases.html)

### [OpenEBS Local PV Device User Guide](/docs/next/uglocalpv-device.html)


<hr>

<br>

