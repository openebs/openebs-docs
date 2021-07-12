---
id: prometheus
title: Using OpenEBS as TSDB for Prometheus
sidebar_label: Prometheus
---
------

<img src="/docs/assets/o-prometheus.png" alt="OpenEBS and Prometheus" style="width:400px;">

## Introduction


Each and every DevOps and SRE's are looking for ease of deployment of their applications in Kubernetes. After successful installation, they will be looking for how easily it can be monitored to maintain the availability of applications in a real-time manner. They can take proactive measures before an issue arise by monitoring the application. Prometheus is the mostly widely used application for scraping cloud native application metrics. Prometheus and OpenEBS together provide a complete open source stack for monitoring.

In this document, we will explain how you can easily set up a monitoring environment in your K8s cluster using Prometheus and use OpenEBS Local PV as the persistent storage for storing the metrics. This guide provides the installation of Prometheus using Helm on dynamically provisioned OpenEBS volumes. 

## Deployment model



<a href="/docs/assets/svg/localpv-device-prometheus-deployment.svg" target="_blank"><img src="/docs/assets/svg/localpv-device-prometheus-deployment.svg" alt="OpenEBS and Prometheus localpv device" style="width:100%;"></a>



We will add 100G of two disks to each node. Disks will be consumed by Prometheus and Alert manager instances using OpenEBS local PV device storage engine. The recommended configuration is to have at least three nodes and two unclaimed external disks to be attached per node. 



## Configuration workflow

1. [Install OpenEBS](/docs/next/prometheus.html#install-openebs)

2. [Select OpenEBS storage engine](/docs/next/prometheus.html#select-openebs-storage-engine)

3. [Configure OpenEBS Local PV StorageClass](/docs/next/prometheus.html#configure-openebs-local-pv-storageclass)

4. [Installing Prometheus Operator](/docs/next/prometheus.html#installing-prometheus-operator)

5. [Accessing Prometheus and Grafana](/docs/next/prometheus.html#accessing-prometheus-and-grafana)

   

### Install OpenEBS

If OpenEBS is not installed in your K8s cluster, this can be done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step.



### Select OpenEBS storage engine

A storage engine is the data plane component of the IO path of a Persistent Volume. In CAS architecture, users can choose different data planes for different application workloads based on a configuration policy. OpenEBS provides different types of storage engines and chooses the right engine that suits your type of application requirements and storage available on your Kubernetes nodes. More information can be read from [here](/docs/next/overview.html#types-of-openebs-storage-engines).

After OpenEBS installation, choose the OpenEBS storage engine as per your requirement. 

- Choose **cStor**, If you are looking for replicated storage feature and other enterprise graded features such as volume expansion, backup and restore, etc. The steps for Prometheus operator installation using OpenEBS cStor storage engine can be obtained from [here](https://github.com/openebs/cstor-operators/blob/master/docs/workload/prometheus-operator/prometheus.md).
- Choose **OpenEBS Local PV**, if you only want to use Prometheus for generating alerts, you will need low latency storage rather than replicated storage.

In this document, we are deploying Prometheus Operator using OpenEBS Local PV device. 



### Configure OpenEBS Local PV StorageClass

There are 2 ways to use OpenEBS Local PV.

- `openebs-hostpath` - Using this option, it will create Kubernetes Persistent Volumes that will store the data into OS host path directory at: `/var/openebs/<"prometheus-pv-name">/`. Select this option, if you don’t have any additional block devices attached to Kubernetes nodes. If you would like to customize the directory where data will be saved, create a new OpenEBS Local PV storage class using the instructions mentioned [here](https://docs.openebs.io/docs/next/uglocalpv-hostpath.html#create-storageclass). 

- `openebs-device` - Using this option, it will create Kubernetes Local PVs using the block devices attached to the node. Select this option when you want to dedicate a complete block device on a node to a Prometheus application pod and other device for Alert manager. You can customize which devices will be discovered and managed by OpenEBS using the instructions [here](/docs/next/uglocalpv-device.html#optional-block-device-tagging). 

The Storage Class `openebs-device` has been chosen to deploy Prometheus Operator in the Kubernetes cluster.

**Note:** Ensure that you have two disks with the required capacity is added to the corresponding nodes prior to Prometheus installation. In this example, we have added two 100G disks to each node.

### Installing Prometheus Operator

In this section, we will install the Prometheus operator. We will later deploy the latest available version of Prometheus application. The following are the high-level overview.

- Label the nodes
- Fetch and update Prometheus repository
- Configure Prometheus Helm `values.yaml`
- Create namespace for installing application
- Install Prometheus operator

#### Label the nodes

Label the nodes with custom label so that Prometheus application will be deployed only on the matched Nodes. Label each node with `node=prometheus`. We have used this label in the Node Affinity for Prometheus and Alert Manager instances. This will ensure to schedule the Prometheus and Alert Manager pods to deploy only on the labelled nodes.

In this example, we used the following command to label our nodes.

```
kubectl label nodes ip-192-168-30-203.ap-south-1.compute.internal node=prometheus
kubectl label nodes ip-192-168-55-47.ap-south-1.compute.internal node=prometheus
kubectl label nodes ip-192-168-68-28.ap-south-1.compute.internal node=prometheus
```

#### Fetch and update Prometheus Helm repository

```
$ helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
$ helm repo update
```

#### Configure Prometheus Helm `values.yaml`

Download `values.yaml` which we will modify before installing Prometheus using Helm.

```none
wget https://raw.githubusercontent.com/prometheus-community/helm-charts/main/charts/kube-prometheus-stack/values.yaml
```

Perform the following changes:

- Update `fullnameOverride: "new" `

- Update `prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues` as `false`

- Update `prometheus.prometheusSpec.replicas` as `3`

- Update `prometheus.prometheusSpec.podAntiAffinity` as `hard`

- Uncomment the following spec in the `values.yaml` for enabling Node Affinity for Prometheus `prometheus.prometheusSpec.affinity` using a custom node label configured in the previous section. Since we used `node=prometheus` for labeling the nodes, mention the same in the Node affinity section for Prometheus deployment.
   ```
   affinity:
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node
              operator: In
              values:
              - prometheus
   ```
   
- Uncomment the following spec in the `values.yaml` for `prometheus.prometheusSpec.storageSpec` and change the StorageClass name of Prometheus with the required StorageClass and required volume capacity. In this case, Storage Class used as `openebs-device` and provided the volume capacity as `90Gi`. Ensure to provide the capacity 
   less than equal to the maximum capacity of the blockdevice, which is going to use it.

   ```
   storageSpec:
     volumeClaimTemplate:
       spec:
         storageClassName: openebs-device
         accessModes: ["ReadWriteOnce"]
         resources:
           requests:
             storage: 90Gi
   ```

- (Optional in case of GKE) Update `prometheusOperator.admissionWebhooks.enabled` as `false`.

- Update `prometheusOperator.tls.enabled` as `false`

- Update `alertmanager.alertmanagerSpec.replicas` as `3`

- Update `alertmanager.alertmanagerSpec.podAntiAffinity` as `hard`

- Uncomment the following spec in the `values.yaml` for enabling Node Affinity for Alert Manager `alertmanager.alertmanagerSpec.affinity` using a custom node label configured in the previous section. Since we used `node=prometheus` for labeling the nodes, mention the same in the Node affinity section for Alert manager.
   ```
   affinity: 
      nodeAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          nodeSelectorTerms:
          - matchExpressions:
            - key: node
              operator: In
              values:
              - prometheus
   ```

- Uncomment the following spec in the `values.yaml` for `alertmanager.alertmanagerSpec.storage` and change the StorageClass name of Alert manager with the required StorageClass name and required volume capacity. In this case, StorageClass used as `openebs-device` and provided the volume capacity as `90Gi`.

  ```none
   storage:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-device
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 90Gi
  ```

#### Create namespace for installing Prometheus operator

```
$ kubectl create ns monitoring
```

#### Install Prometheus operator

The following command will install both Prometheus and Grafana components.
```
$ helm install prometheus prometheus-community/kube-prometheus-stack --namespace monitoring -f values.yaml
```
**Note:** Check compatibility for your Kubernetes version and Prometheus stack from [here](https://github.com/prometheus-operator/kube-prometheus/tree/release-0.7#compatibility).



Verify Prometheus related pods are installed under **monitoring** namespace

```
$ kubectl get pod -n monitoring

NAME                                             READY   STATUS    RESTARTS   AGE
alertmanager-new-alertmanager-0                  2/2     Running   0          67m
alertmanager-new-alertmanager-1                  2/2     Running   0          67m
alertmanager-new-alertmanager-2                  2/2     Running   0          67m
new-operator-965786d6b-tln89                     1/1     Running   0          67m
prometheus-grafana-6549f869b5-82hdc              2/2     Running   0          67m
prometheus-kube-state-metrics-685b975bb7-wr28n   1/1     Running   0          67m
prometheus-new-prometheus-0                      2/2     Running   1          67m
prometheus-new-prometheus-1                      2/2     Running   1          67m
prometheus-new-prometheus-2                      2/2     Running   1          67m
prometheus-prometheus-node-exporter-246kl        1/1     Running   0          67m
prometheus-prometheus-node-exporter-6pqcj        1/1     Running   0          67m
prometheus-prometheus-node-exporter-gdzhc        1/1     Running   0          67m
```

Verify Prometheus related PVCs are created under **monitoring** namespace

```
$ kubectl get pvc -n monitoring

NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-0   Bound    pvc-ad916e53-778e-448f-95dd-07136569e064   90Gi       RWO            openebs-device   67m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-1   Bound    pvc-095772d7-c5f8-43f0-b0b0-f546d75bf3a2   90Gi       RWO            openebs-device   67m
alertmanager-new-alertmanager-db-alertmanager-new-alertmanager-2   Bound    pvc-e36e26eb-2d45-4f01-8aab-7e90f51d5252   90Gi       RWO            openebs-device   67m
prometheus-new-prometheus-db-prometheus-new-prometheus-0           Bound    pvc-471edd10-6037-4a30-9451-5417e6caf5bd   90Gi       RWO            openebs-device   67m
prometheus-new-prometheus-db-prometheus-new-prometheus-1           Bound    pvc-c1ef4a1f-d3c0-404b-ad78-c6f5eb94bfab   90Gi       RWO            openebs-device   67m
prometheus-new-prometheus-db-prometheus-new-prometheus-2           Bound    pvc-36a78cfe-b86b-4e7d-b1a1-3bd70067fc47   90Gi       RWO            openebs-device   67m
```
Verify Prometheus related services created under **monitoring** namespace
```
$ kubectl get svc -n monitoring

NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                 ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   67m
new-alertmanager                      ClusterIP   10.100.190.160   <none>        9093/TCP                     68m
new-operator                          ClusterIP   10.100.22.183    <none>        8080/TCP                     68m
new-prometheus                        ClusterIP   10.100.31.79     <none>        9090/TCP                     68m
prometheus-grafana                    ClusterIP   10.100.149.205   <none>        80/TCP                       68m
prometheus-kube-state-metrics         ClusterIP   10.100.21.186    <none>        8080/TCP                     68m
prometheus-operated                   ClusterIP   None             <none>        9090/TCP                     67m
prometheus-prometheus-node-exporter   ClusterIP   10.100.69.43     <none>        9100/TCP                     68m
```



For ease of simplicity in testing the deployment, we are going to use NodePort for **prometheus-kube-prometheus-prometheus** and **prometheus-grafana** services . Please be advised to consider using **LoadBalancer** or **Ingress**, instead of **NodePort**, for production deployment.



Change Prometheus service to **NodePort** from **ClusterIP**:

```
$ kubectl patch svc new-prometheus -n monitoring -p '{"spec": {"type": "NodePort"}}'
```



Change prometheus-grafana service to **LoadBalancer**/**NodePort** from **ClusterIP**:

```
$ kubectl patch svc prometheus-grafana -n monitoring -p '{"spec": {"type": "NodePort"}}'
```

**Note:** If the user needs to access Prometheus and Grafana outside the network, the service type can be changed or a new service should be added to use LoadBalancer or create Ingress resources for production deployment.

Sample output after making the above 2 changes in services:

```
$ kubectl get svc -n monitoring

NAME                                  TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
alertmanager-operated                 ClusterIP   None             <none>        9093/TCP,9094/TCP,9094/UDP   69m
new-alertmanager                      ClusterIP   10.100.190.160   <none>        9093/TCP                     69m
new-operator                          ClusterIP   10.100.22.183    <none>        8080/TCP                     69m
new-prometheus                        NodePort    10.100.31.79     <none>        9090:31669/TCP               69m
prometheus-grafana                    NodePort    10.100.149.205   <none>        80:30253/TCP                 69m
prometheus-kube-state-metrics         ClusterIP   10.100.21.186    <none>        8080/TCP                     69m
prometheus-operated                   ClusterIP   None             <none>        9090/TCP                     69m
prometheus-prometheus-node-exporter   ClusterIP   10.100.69.43     <none>        9100/TCP                     69m
```



### Accessing Prometheus and Grafana

Get the node details using the following command:

```
$ kubectl get node -o wide

ip-192-168-30-203.ap-south-1.compute.internal   Ready    <none>   5h13m   v1.19.5   192.168.30.203   52.66.223.37    Ubuntu 20.04.2 LTS   5.4.0-1037-aws   docker://19.3.8
ip-192-168-55-47.ap-south-1.compute.internal    Ready    <none>   5h13m   v1.19.5   192.168.55.47    35.154.152.99   Ubuntu 20.04.2 LTS   5.4.0-1037-aws   docker://19.3.8
ip-192-168-68-28.ap-south-1.compute.internal    Ready    <none>   5h13m   v1.19.5   192.168.68.28    65.0.131.40     Ubuntu 20.04.2 LTS   5.4.0-1037-aws   docker://19.3.8
```




Verify Prometheus service is accessible over web browser using http://<any_node_external-ip:<NodePort>

Example:

http://52.66.223.37:31669

**Note**: It may be required to allow the Node Port number/traffic in the Firewall/Security Groups to access the above Grafana and Prometheus URL on the web browser.




Launch Grafana using Node's External IP and with corresponding NodePort of **prometheus-grafana** service

http://<any_node_external-ip>:<Grafana_SVC_NodePort>

Example:

http://52.66.223.37:30253



Grafana Credentials:

**Username**: admin

**Password**: prom-operator



Password can be obtained using the command

```
$ (kubectl get secret \
    --namespace monitoring prometheus-grafana \
    -o jsonpath="{.data.admin-password}" \
    | base64 --decode ; echo
)
```

The above credentials need to be provided when you need to access the Grafana console. Login to your Grafana console using the above credentials.

Users can upload a Grafana dashboard for Prometheus in 3 ways. 

- First method is by proving the Grafana id of the corresponding dashboard and then load it.
  Just find the Grafana dashboard id of the Prometheus and then just mention this id and then load it. The Grafana dashboard id of Prometheus is **3681**.
  
- Another approach is download the following Grafana dashboard JSON file for Prometheus and then paste it in the console and then load it. 

  ```
  https://raw.githubusercontent.com/FUSAKLA/Prometheus2-grafana-dashboard/master/dashboard/prometheus2-dashboard.json

- The other way to monitor Prometheus Operator is by using the inbuilt Prometheus dashboard. This can be obtained by searching on the Grafana dashboard and finding the Prometheus dashboard under the General category.


<br>

## See Also:

### [OpenEBS use cases](/docs/next/usecases.html)

### [Understanding NDM](/docs/next/ugndm.html)

### [Local PV concepts](/docs/next/localpv.html)

### [Local PV User guide](/docs/next/uglocalpv-device.html)

<br>

<hr>