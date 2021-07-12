---
id: elasticsearch
title: OpenEBS for Elasticsearch
sidebar_label: Elasticsearch
---
------

<img src="/docs/assets/o-elastic.png" alt="OpenEBS and Elasticsearch" style="width:400px;">

<br>

## Introduction

<br>

EFK is the most popular cloud native logging solution on Kubernetes for On-Premise as well as cloud platforms. In the EFK stack, Elasticsearch is a stateful application that needs persistent storage. Logs of production applications need to be stored for a long time which requires reliable and highly available storage.  OpenEBS and EFK together provides a complete logging solution.

This guide explains the basic installation for Elasticsearch operators on OpenEBS Local PV devices using KUDO. We will be installing Fluentd and Kibana to form the EFK stack.


Advantages of using OpenEBS LocalPV for Elasticsearch database:

- All the logs data is stored locally and managed natively to Kubernetes
- Low latency and better performance

<br>


## Deployment model

<br>
<a href="/docs/assets/svg/Local-PV-Devices-elastic-deployment.svg" target="_blank"><img src="/docs/assets/svg/Local-PV-Devices-elastic-deployment.svg" alt="OpenEBS and Elasticsearch" style="width:100%;"></a>

The Local PV volume will be provisioned on a node where Elasticsearch components are getting scheduled and uses one of the matching unclaimed block device for each of them, which will then use the entire block device for storing data. No other application can use this device. If users have limited blockdevices attached to some nodes, they can use `nodeSelector` in the application YAML to provision applications on particular nodes where the available block device is present. The recommended configuration is to have at least three nodes and two unclaimed external disk to be attached per node. 

The Elasticsearch deployment has the following components, which will use the OpenEBS LocalPV Devices for storage:
1. coordinator pod: 1
2. master pods: 3
3. data pods: 2 


*Note: Elasticsearch can be deployed both as `deployment` or as `statefulset`. When Elasticsearch deployed as `statefulset`, you don't need to replicate the data again at OpenEBS level. When Elasticsearch is deployed as `deployment`, consider 3 OpenEBS replicas, choose the StorageClass accordingly.*
<br>

## Configuration workflow

1. [Install OpenEBS](/docs/next/elasticsearch.html#install-openebs)
2. [Select OpenEBS storage engine](/docs/next/elasticsearch.html#select-openebs-storage-engine)
3. [Configure OpenEBS Local PV StorageClass](/docs/next/elasticsearch.html#configure-openebs-local-pv-storageclass)
4. [Installing KUDO Operator](/docs/next/elasticsearch.html#installing-kudo-operator)
5. [Installing and Accessing Elasticsearch](/docs/next/elasticsearch.html#installing-and-accessing-elasticsearch)
6. [Installing Kibana](/docs/next/elasticsearch.html#installing-kibana)
7. [Installing Fluentd-ES](/docs/next/elasticsearch.html#installing-fluentd-es)



### Install OpenEBS

If OpenEBS is not installed in your K8s cluster, this can be done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step. 


### Select OpenEBS storage engine

A storage engine is the data plane component of the IO path of a Persistent Volume. In CAS architecture, users can choose different data planes for different application workloads based on a configuration policy. OpenEBS provides different types of storage engines and you should choose the right engine that suits your type of application requirements and storage available on your Kubernetes nodes. More information can be read from [here](/docs/next/overview.html#types-of-openebs-storage-engines).

After OpenEBS installation, choose the OpenEBS storage engine as per your requirement. 

- Choose **cStor**, If you are looking for replicated storage feature and other enterprise graded features such as volume expansion, backup and restore, etc. The steps for Elasticsearch installation using OpenEBS cStor storage engine can be obtained from [here](https://github.com/openebs/cstor-operators/blob/master/docs/workload/elasticsearch/elasticsearch.md).

- Choose **OpenEBS Local PV**, If you are looking for direct attached storage or low latency data write or if the application manages data replication.

In this document, we are deploying Elasticsearch using OpenEBS Local PV. 

### Configure OpenEBS Local PV StorageClass

Depending on the type of storage attached to your Kubernetes worker nodes, you can select from different flavors of Dynamic Local PV - Hostpath, Device, LVM, ZFS or Rawfile. For more information you can read [here](/docs/next/localpv.html).

The Storage Class `openebs-device` has been chosen to deploy Elasticsearch in the Kubernetes cluster.

**Note:** Ensure that you have two disks with the required capacity added to the corresponding nodes prior to Elasticsearch installation. In this example, we have added two 100G disks to each node.


###  Installing KUDO Operator

In this section, we will install the KUDO operator. We will later deploy the latest available version of Elasticsearch using KUDO. 

Use the latest stable version of KUDO CLI. The latest version of KUDO can be checked from [here](https://github.com/kudobuilder/kudo/releases).

#### Verify if Cert-manager is installed

For installing KUDO operator, the Cert-manager must be already installed in your cluster. If not, install the Cert-manager. The instruction can be found from [here](https://cert-manager.io/docs/installation/kubernetes/#installing-with-regular-manifests). Since our K8s version is v1.18.12, we have installed Cert-manager using the following command.

```bash
kubectl apply -f https://github.com/jetstack/cert-manager/releases/download/v1.2.0/cert-manager.yaml
```

```bash
kubectl get pods --namespace cert-manager
```
```bash
#sample output
NAME                                      READY   STATUS    RESTARTS   AGE
cert-manager-7747db9d88-7qjnm             1/1     Running   0          81m
cert-manager-cainjector-87c85c6ff-whnzr   1/1     Running   0          81m
cert-manager-webhook-64dc9fff44-qww8s     1/1     Running   0          81m
```

#### Installing KUDO operator into cluster

Once prerequisites are installed you need to initialize the KUDO operator. The following command will install KUDO v0.18.2.

```bash
kubectl-kudo init --version 0.18.2
```
```bash
#sample output
$KUDO_HOME has been configured at /home/k8s/.kudo
✅ installed crds
✅ installed namespace
✅ installed service account
✅ installed webhook
✅ installed kudo controller
```

Verify pods in the `kudo-system` namespace:
```bash
kubectl get pod -n kudo-system
```
```bash
#sample output
NAME                        READY   STATUS    RESTARTS   AGE
kudo-controller-manager-0   1/1     Running   0          25s
```

#### Setting OpenEBS Storage Class as default

Change the default storage class from your current setting to OpenEBS LocalPV Device. For example, in this tutorial default storage class is used as `openebs-device` from standard.
```bash
kubectl patch storageclass standard -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
kubectl patch storageclass openebs-device -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

#### Verify default Storage Class
 
List the storage classes and verify `openebs-device` is set to `default`.
```bash
kubectl get sc
```
```bash
#sample output
NAME                        PROVISIONER                                                RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
openebs-device (default)    openebs.io/local                                           Delete          WaitForFirstConsumer   false                  4h30m
openebs-hostpath            openebs.io/local                                           Delete          WaitForFirstConsumer   false                  4h30m
openebs-jiva-default        openebs.io/provisioner-iscsi                               Delete          Immediate              false                  4h30m
openebs-snapshot-promoter   volumesnapshot.external-storage.k8s.io/snapshot-promoter   Delete          Immediate              false                  4h30m
premium-rwo                 pd.csi.storage.gke.io                                      Delete          WaitForFirstConsumer   true                   5h13m
standard                    kubernetes.io/gce-pd                                       Delete          Immediate              true                   5h13m
standard-rwo                pd.csi.storage.gke.io                                      Delete          WaitForFirstConsumer   true                   5h13
```

### Installing and Accessing Elasticsearch

Set instance and namespace variables:
```bash
export instance_name=elastic
export namespace_name=default
kubectl-kudo install elastic --namespace=$namespace_name --instance $instance_name
```
```bash
#sample output
operator default/elastic created
operatorversion default/elastic-7.0.0-0.2.1 created
instance default/elastic created
```
#### Verifying Elastic pods

```bash
kubectl get pods -n $namespace_name
```
```bash
#sample output
NAME                    READY   STATUS    RESTARTS   AGE
elastic-coordinator-0   1/1     Running   0          31s
elastic-data-0          1/1     Running   0          56s
elastic-data-1          1/1     Running   0          44s
elastic-master-0        1/1     Running   0          2m31s
elastic-master-1        1/1     Running   0          119s
elastic-master-2        1/1     Running   0          90s
```

#### Verifying Services

```bash
kubectl get svc -n $namespace_name
```
```bash
#sample output
NAME                     TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
elastic-coordinator-hs   ClusterIP   None         <none>        9200/TCP   62s
elastic-data-hs          ClusterIP   None         <none>        9200/TCP   87s
elastic-ingest-hs        ClusterIP   None         <none>        9200/TCP   50s
elastic-master-hs        ClusterIP   None         <none>        9200/TCP   3m2s
kubernetes               ClusterIP   10.48.0.1    <none>        443/TCP    5h18m
```


#### Verifying Elastic instance status

```bash
kubectl kudo plan status --namespace=$namespace_name \
 --instance $instance_name
```
```bash
#sample output
Plan(s) for "elastic" in namespace "default":
.
└── elastic (Operator-Version: "elastic-7.0.0-0.2.1" Active-Plan: "deploy")
    └── Plan deploy (serial strategy) [COMPLETE], last updated 2021-02-22 16:18:17
        ├── Phase deploy-master (parallel strategy) [COMPLETE]
        │   └── Step deploy-master [COMPLETE]
        ├── Phase deploy-data (parallel strategy) [COMPLETE]
        │   └── Step deploy-data [COMPLETE]
        ├── Phase deploy-coordinator (parallel strategy) [COMPLETE]
        │   └── Step deploy-coordinator [COMPLETE]
        └── Phase deploy-ingest (parallel strategy) [COMPLETE]
            └── Step deploy-ingest [COMPLETE]
```

#### Accessing Elasticsearch

Enter into one of the master pod using exec command:
```bash
kubectl exec -it elastic-master-0 -- bash
```
```bash
#sample output
[root@elastic-master-0 elasticsearch]#
```
Run below command inside Elastic master pod:
```bash
curl -X POST "elastic-coordinator-hs:9200/twitter/_doc/" -H 'Content-Type: application/json' -d'
 {
     "user" : "openebs",
     "post_date" : "2021-03-02T14:12:12",
     "message" : "Test data entry"
 }'
```
Following is the output of the above command:
```bash
#sample output
{"_index":"twitter","_type":"_doc","_id":"LoliyXcBg9iVzVnOj5QL","_version":1,"result":"created","_shards":{"total":2,"successful":1,"failed":0},"_seq_no":0,"_primary_term":1}
[root@elastic-master-0 elasticsearch]#
```

The above command added data into Elasticsearch. You can use the following command to query for the inserted data:
```bash
curl -X GET "elastic-coordinator-hs:9200/twitter/_search?q=user:openebs&pretty"
```
```bash
#sample output
{
  "took" : 141,
  "timed_out" : false,
  "_shards" : {
    "total" : 1,
    "successful" : 1,
    "skipped" : 0,
    "failed" : 0
  },
  "hits" : {
    "total" : {
      "value" : 1,
      "relation" : "eq"
    },
    "max_score" : 0.2876821,
    "hits" : [
      {
        "_index" : "twitter",
        "_type" : "_doc",
        "_id" : "qoWznXkBpvRSYN1fECvZ",
        "_score" : 0.2876821,
        "_source" : {
          "user" : "openebs",
          "post_date" : "2021-03-02T14:12:12",
          "message" : "Test data entry"
        }
      }
    ]
  }
}
```

Now, let's get the details of Elasticsearch cluster. The cluster information will show the Elasticsearch version, cluster name and other details. If you are getting similar information, then it means your Elasticsearch deployment is successful.

```bash
curl localhost:9200
```
```bash
#sample output
{
  "name" : "elastic-master-0",
  "cluster_name" : "elastic-cluster",
  "cluster_uuid" : "A0qErYmCS2OpJtmgR_j3ow",
  "version" : {
    "number" : "7.10.1",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "1c34507e66d7db1211f66f3513706fdf548736aa",
    "build_date" : "2020-12-05T01:00:33.671820Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```


### Installing Kibana

First, add helm repository of Elastic.
```bash
helm repo add elastic https://helm.elastic.co && helm repo update
```

Install Kibana deployment using `helm` command. Ensure to meet required prerequisites corresponding to your helm version. Fetch the Kibana `values.yaml`:
```bash
wget https://raw.githubusercontent.com/elastic/helm-charts/master/kibana/values.yaml
``` 

Edit the following parameters:

1. `elasticsearchHosts` as `"http://elastic-coordinator-hs:9200"` # service name of Elastic search.
2. `service.type` as `"NodePort"`.
3. `service.nodePort` as `"30295"` # since this port is already added in our network firewall rules.
4. `imageTag` as `"7.10.1"` , it should be the same image tag of Elasticsearch. In our case,  Elasticsearch image tag is 7.10.1.


Now install Kibana using Helm:
```bash
helm install  kibana -f values.yaml elastic/kibana
```

Verifying Kibana Pods and Services:
```bash
kubectl get pod
```
```bash
#sample output
NAME                             READY   STATUS    RESTARTS   AGE
elastic-coordinator-0            1/1     Running   0          12m
elastic-data-0                   1/1     Running   0          12m
elastic-data-1                   1/1     Running   0          12m
elastic-master-0                 1/1     Running   0          11m
elastic-master-1                 1/1     Running   0          11m
elastic-master-2                 1/1     Running   0          12m
kibana-kibana-74cbc4d654-h8djr   1/1     Running   0          6m33s
```

```bash
kubectl get svc
```
```bash
#sample output
NAME                     TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
elastic-coordinator-hs   ClusterIP   None           <none>        9200/TCP         18m
elastic-data-hs          ClusterIP   None           <none>        9200/TCP         19m
elastic-ingest-hs        ClusterIP   None           <none>        9200/TCP         18m
elastic-master-hs        ClusterIP   None           <none>        9200/TCP         20m
kibana-kibana            NodePort    10.48.12.146   <none>        5601:30295/TCP   7m1s
kubernetes               ClusterIP   10.48.0.1      <none>        443/TCP          5h36m
```



### Installing Fluentd-ES

Fetch the `values.yaml`:
```bash
wget https://raw.githubusercontent.com/bitnami/charts/master/bitnami/fluentd/values.yaml
helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
```

Replace the following section in the `values.yaml` file with new content.

Old:
```yaml   
     # Send the logs to the standard output
      <match **>
        @type stdout
      </match>
```
New:
```yaml
      # Send the logs to the elasticsearch output
      <match **>
        @type elasticsearch
        include_tag_key true
        host elastic-coordinator-hs
        port 9200
        logstash_format true

        <buffer>
          @type file
          path /opt/bitnami/fluentd/logs/buffers/logs.buffer
          flush_thread_count 2
          flush_interval 5s
        </buffer>
      </match>
 ```
 
 
 
Install Fluentd-Elasticsearch DaemonSet using the new values:
```bash
helm install fluentd -f values.yaml bitnami/fluentd
```

Verify Fluentd Daemonset, Pods and Services:
```bash
kubectl get ds
```
```bash
#sample output
NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
fluentd   3         3         3       3            3           <none>          74m
```
```bash
kubectl get pod
```
```bash
#sample output
NAME                                 READY   STATUS    RESTARTS   AGE
pod/elastic-coordinator-0            1/1     Running   0          67m
pod/elastic-data-0                   1/1     Running   0          66m
pod/elastic-data-1                   1/1     Running   0          67m
pod/elastic-master-0                 1/1     Running   0          66m
pod/elastic-master-1                 1/1     Running   0          66m
pod/elastic-master-2                 1/1     Running   0          66m
pod/fluentd-0                        1/1     Running   0          106s
pod/fluentd-4sbs4                    1/1     Running   0          106s
pod/fluentd-5mvv9                    1/1     Running   2          106s
pod/fluentd-z2sxt                    1/1     Running   2          106s
pod/kibana-kibana-74cbc4d654-h8djr   1/1     Running   0          9m46s
```

```bash
kubectl get svc
```
```bash
#sample output
elastic-coordinator-hs   ClusterIP   None           <none>        9200/TCP             67m
elastic-data-hs          ClusterIP   None           <none>        9200/TCP             66m
elastic-ingest-hs        ClusterIP   None           <none>        9200/TCP             67m
elastic-master-hs        ClusterIP   None           <none>        9200/TCP             66m
fluentd-aggregator       ClusterIP   10.48.13.214   <none>        9880/TCP,24224/TCP   106s
fluentd-forwarder        ClusterIP   10.48.7.6      <none>        9880/TCP             106s
fluentd-headless         ClusterIP   None           <none>        9880/TCP,24224/TCP   106s
kibana-kibana            NodePort    10.48.12.146   <none>        5601:30295/TCP       9m46s
kubernetes               ClusterIP   10.48.0.1      <none>        443/TCP              5h40m
```



Getting logs from the indices:

1. Goto Kibana dashboard.
2. Click on Management->Stack Management which is placed at left side bottom most.
3. Click on index patterns listed under Kibana and then click on `Create Index pattern`.
4. Provide `logstash-*` inside the index pattern box and then select `Next step`.
5. In the next step, inside the `Time Filter` field name, select the `@timestamp` field from the dropdown menu, and click `Create index pattern`.
6. Now click on the `Discover` button listed on the top left of the side menu bar.
7. There will be a dropdown menu where you can select the available indices.
8. In this case, you have to select `logstash-*` from the dropdown menu.


Now let's do some tests:

If you want to get the logs of NDM pods, type the following text inside the `Filters` field.
`kubernetes.labels.openebs_io/component-name.keyword : "ndm"` and then choose the required date and time period. After that, click Apply.
You will see the OpenEBS NDM pod logs listed on the page.




<br>

## See Also:

### [OpenEBS use cases](/docs/next/usecases.html)

### [Understanding NDM](/docs/next/ugndm.html)

### [Local PV concepts](/docs/next/localpv.html)

### [Local PV User guide](/docs/next/uglocalpv-device.html)


<br>

<hr>
<br>
