---
id: nuodb
title: OpenEBS for NuoDB
sidebar_label: NuoDB
---
------

<img src="/docs/assets/o-nuodb.png" alt="OpenEBS and Nuodb" style="width:400px;">

<br>

## Introduction

NuoDB’s distributed SQL database combines the elastic scale and continuous availability of the cloud with the transactional consistency and durability that databases of record demand. NuoDB is deployed usually as a `StatefulSet` on Kubernetes and requires persistent storage for each instance of NuoDB StorageManager instance. OpenEBS provides persistent volumes on the fly when StorageManagers are scaled up.

<br>

**Advantages of using OpenEBS for NuoDB database:**

- No need to manage the local disks, they are managed by OpenEBS
- Large size PVs can be provisioned by OpenEBS and NuoDB
- Start with small storage and add disks as needed on the fly. Sometimes NuoDB instances are scaled up because of capacity on the nodes. With OpenEBS persistent volumes, capacity can be thin provisioned and disks can be added to OpenEBS on the fly without disruption of service 
- If required, take backup of the NuoDB data periodically and back them up to S3 or any object storage so that restoration of the same data is possible to the same or any other Kubernetes cluster

<br>

*Note: NuoDB can be deployed both as `deployment` or as `statefulset`. When NuoDB deployed as `statefulset`, you don't need to replicate the data again at OpenEBS level. When NuoDB is deployed as `deployment`, consider 3 OpenEBS replicas, choose the StorageClass accordingly.*

<br>

<hr>

<br>





## Deployment model



<br>



<a href="/docs/assets/svg/nuodb-deployment.svg" target="_blank"><img src="/docs/assets/svg/nuodb-deployment.svg" alt="OpenEBS and NuoDB" style="width:100%;"></a>

<br>

<hr>

<br>



## Configuration workflow

1. **Install OpenEBS**

   If OpenEBS is not installed in your K8s cluster, this can done from [here](/docs/next/installation.html). If OpenEBS is already installed, go to the next step. 

2. **Configure cStor Pool**

   After OpenEBS installation, cStor pool has to be configured. If cStor Pool is not configured in your OpenEBS cluster, this can be done from [here](/docs/next/ugcstor.html#creating-cStor-storage-pools). Sample YAML named **openebs-config.yaml** for configuring cStor Pool is provided in the Configuration details below. During cStor Pool creation, make sure that the maxPools parameter is set to >=3. If cStor pool is already configured, go to the next step. 

4. **Create Storage Class**

   You must configure a StorageClass to provision cStor volume on given cStor pool. StorageClass is the interface through which most of the OpenEBS storage policies are defined. In this solution we are using a StorageClass to consume the cStor Pool which is created using external disks attached on the Nodes.  Since NuoDB is a StatefulSet application, it requires only single storage replica. So cStor volume `replicaCount` is =1. Sample YAML named **openebs-sc-disk.yaml** to consume cStor pool with cStor volume replica count as 1 is provided in the configuration details below.

5. Download the YAML spec files from OpenEBS litmus repository. 

   ```
   wget https://raw.githubusercontent.com/openebs/litmus/master/apps/nuodb/deployers/nuodb.yaml
   
   wget https://raw.githubusercontent.com/openebs/litmus/master/apps/nuodb/deployers/nuodb-sm.yaml
   
   wget https://raw.githubusercontent.com/openebs/litmus/master/apps/nuodb/deployers/nuodb-te.yaml
   ```

6. Edit the above YAML spec files and update them with correct StorageClass. For example, replace `openebs-cstor-sparse`  with  `openebs-cstor-disk`

7. **Launch and test NuoDB**

   Create a namespace called **testns** and  apply following YAML files to deploy NuoDB application. Sample YAML files are provided in the Configuration details below.

   ```
   kubectl apply -f nuodb.yaml -n testns
   kubectl apply -f nuodb-sm.yaml -n testns
   kubectl apply -f nuodb-te.yaml -n testns
   ```



<br>

<hr>

<br>





## Reference at [openebs.ci](https://openebs.ci/)

Deployment YAML spec files for NuoDB and OpenEBS resources are found [here](https://github.com/openebs/litmus/blob/master/apps/nuodb/deployers/nuodb.yaml)

[OpenEBS-CI dashboard of NuoDB](https://openebs.ci/nuodb-cstor)

[Live access to NuoDB dashboard](https://insights.nuodb.com/3N5YV375G0/)



<br>

<hr>

<br>



## Post deployment Operations

<br>



**Monitor OpenEBS Volume size** 

It is not seamless to increase the cStor volume size (refer to the roadmap item). Hence, it is recommended that sufficient size is allocated during the initial configuration.

**Monitor cStor Pool size**

As in most cases, cStor pool may not be dedicated to just NuoDB database alone. It is recommended to watch the pool capacity and add more disks to the pool before it hits 80% threshold. See [cStorPool metrics](/docs/next/ugcstor.html#monitor-pool) 



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

**nuodb.yaml**

```yaml
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    description: |
      Headless service permitting us to retrieve DNS A records that point directly to the pods backing the service.
    "service.alpha.kubernetes.io/tolerate-unready-endpoints": "true"
  name: domain
  labels:
    app: nuodb
    group: nuodb
spec:
  clusterIP: None
  ports:
  - { name: 8888-tcp,   port: 8888,   protocol: TCP,  targetPort: 8888  }
  - { name: 48004-tcp,  port: 48004,  protocol: TCP,  targetPort: 48004 }
  - { name: 48005-tcp,  port: 48005,  protocol: TCP,  targetPort: 48005 }
  selector:
    dns: domain
  sessionAffinity: None
  type: ClusterIP
status:
  loadBalancer: {}

---
apiVersion: v1
kind: Service
metadata:
  annotations:
    description: "Service (and load-balancer) for Admin pods."
  labels:
    app: nuodb
    group: nuodb
  name: admin
spec:
  ports:
  - { name: 8888-tcp,   port: 8888,   protocol: TCP,  targetPort: 8888  }
  - { name: 48004-tcp,  port: 48004,  protocol: TCP,  targetPort: 48004 }
  - { name: 48005-tcp,  port: 48005,  protocol: TCP,  targetPort: 48005 }
  selector:
    app: admin
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}

---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: admin
  labels:
    app: nuodb
    group: nuodb
spec:
  serviceName: domain
  replicas: 1
  selector:
    matchLabels:
      affinity: admin
  template:
    metadata:
      labels:
        dns: domain
        app: admin
        affinity: admin
    spec:
      securityContext:
        runAsUser: 0
        fsGroup: 0
      containers:
      - name: admin
        image: nuodb/nuodb-ce:latest
        imagePullPolicy: IfNotPresent
        ports:
        - { containerPort: 8888,  protocol: TCP }
        - { containerPort: 48004, protocol: TCP }
        - { containerPort: 48005, protocol: TCP }
        resources:
          requests:
            memory: "512Mi"
            cpu: "0.5"
          limits:
            memory: "1024Mi"
            cpu: "1"
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - { name: NUODB_DOMAIN_ENTRYPOINT,      value: "admin-0.domain" }
        - { name: NUODB_ALT_ADDRESS,            value: "$(POD_NAME).domain.testns.svc" }
        - { name: NUODB_VARDIR,                 value: "/var/opt/nuodb/$(POD_NAME).testns" }
        args: [ "nuoadmin" ]
        livenessProbe:
          initialDelaySeconds: 30
          periodSeconds: 15
          tcpSocket:
            port: 8888
        readinessProbe:
          initialDelaySeconds: 30
          periodSeconds: 15
          exec:
            command: [ "nuodocker", "check", "servers" ]
        volumeMounts:
        - name: raftlog
          mountPath: /var/opt/nuodb
        - name: log-volume
          mountPath: /var/log/nuodb
      volumes:
        - name: log-volume
          emptyDir: {}
  volumeClaimTemplates:
  - metadata:
      name: raftlog
      annotations:
        volume.beta.kubernetes.io/storage-class: openebs-cstor-sparse
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5G
---
apiVersion: v1
kind: Service
metadata:
  annotations:
    description: "Service for redirect."
  labels:
    app: insights
    group: nuodb
  name: insights-server
spec:
  ports:
  - { name: 8080-tcp,   port: 8080,   protocol: TCP,  targetPort: 8080  }
  selector:
    app: insights
    group: nuodb
  sessionAffinity: None
  type: LoadBalancer
status:
  loadBalancer: {}

---
apiVersion: v1
kind: ConfigMap
metadata:
  annotations:
    description: "Config map for metrics input and insights output"
  name: insights-configmap
  labels:
    group: nuodb
data:
  nuoca.yml.template: |
    ---
    NUOCA_LOGFILE: /var/log/nuodb/nuoca.log

    INPUT_PLUGINS:
    - NuoAdminNuoMon:
       description : Collection from NuoDB engines
       nuocaCollectionName: NuoMon
       api_server: https://domain:8888
       client_key:  /etc/nuodb/keys/nuocmd.pem
    OUTPUT_PLUGINS:
    - RestClient:
        url: ${INSIGHTS_INGEST_URL}
  nuoinsights: |
    #!/bin/bash
    . ${NUODB_HOME}/etc/nuodb_setup.sh
    : ${NUOCMD_API_SERVER:=https://domain:8888}

    SCRIPT=$(cat <<EOF
    from BaseHTTPServer import BaseHTTPRequestHandler,HTTPServer
    from nuoca import NuoCA as nuoca
    import nuodb_mgmt

    PORT_NUMBER = 8080

    def get_insights_dashboard():
      conn = nuodb_mgmt.AdminConnection('${NUOCMD_API_SERVER}', '/etc/nuodb/keys/nuocmd.pem')
      metadata = nuoca.get_insights_metadata(conn)
      return metadata.get(nuoca.INSIGHTS_DASHBOARD_KEY,None)

      #ingest_url    = metadata[nuoca.INSIGHTS_URL_KEY]
      #subscriber_id = metadata[nuoca.INSIGHTS_ID_KEY]
      #return ingest_url[:-6] + subscriber_id

    class myHandler(BaseHTTPRequestHandler):
      def do_GET(self):
        new_path = get_insights_dashboard()
        if new_path and new_path != "None":
          self.send_response(307)
          self.send_header('Location', new_path)
          self.end_headers()
        else:
          self.send_response(404)
        self.end_headers()
        return

    try:
      server = HTTPServer(('', PORT_NUMBER), myHandler)
      print 'Started httpserver on port ' , PORT_NUMBER

      #Wait forever for incoming http requests
      server.serve_forever()

    except KeyboardInterrupt:
      print '^C received, shutting down the web server'
      server.socket.close()

    EOF
    )

    export PYTHONPATH=/opt/nuodb/drivers/pynuoadmin
    exec /opt/nuodb/etc/python/nuopython -c "${SCRIPT}"

---
apiVersion: v1
kind: Pod
metadata:
  name: nuodb-insights
  labels:
    app: insights
    group: nuodb
spec:
  initContainers:
  - name: optin
    image: nuodb/nuodb-ce:latest
    imagePullPolicy: IfNotPresent
    args:
    - "/bin/sh"
    - "-c"
    - "[ \"${OPT_IN}\" == \"true\" ] && nuoca enable insights --connect-timeout 300 || nuoca disable insights --connect-timeout 300"
    env:
    - { name: NUOCMD_API_SERVER , value: "https://domain:8888" }
  containers:
  - name: insights
    image: nuodb/nuodb-ce:latest
    imagePullPolicy: IfNotPresent
    args: [ "nuoca", "start" , "nuoca", "--insights" ]
    env:
    - { name: NUOCMD_API_SERVER , value: "https://domain:8888" }
    volumeMounts:
    - name: log-volume
      mountPath: /var/log/nuodb
    - name: config-insights
      mountPath: /etc/nuodb/nuoca.yml.template
      subPath: nuoca.yml.template
  - name: insights-server
    image: nuodb/nuodb-ce:latest
    imagePullPolicy: IfNotPresent
    args: [ "nuoinsights" ]
    ports:
    - { name: 8080-tcp,  containerPort: 8080,  protocol: TCP }
    env:
    - { name: NUOCMD_API_SERVER , value: "https://domain:8888" }
    volumeMounts:
      - name: log-volume
        mountPath: /var/log/nuodb
      - name: nuoinsights
        mountPath: /usr/local/bin/nuoinsights
        subPath: nuoinsights
  volumes:
  - name: log-volume
    emptyDir: {}
  - name: config-insights
    configMap:
      name: insights-configmap
  - name: nuoinsights
    configMap:
      name: insights-configmap
      defaultMode: 0754
```

**nuodb-sm.yaml**

```
---
apiVersion: apps/v1beta1
kind: StatefulSet
metadata:
  name: sm
  labels:
    app: nuodb
    group: nuodb
spec:
  serviceName: sm
  replicas: 1
  selector:
    matchLabels:
      nodetype: sm
      database: nuodb
      app: sm
      group: nuodb
  podManagementPolicy: Parallel
  template:
    metadata:
      labels:
        nodetype: sm
        database: nuodb
        app: sm
        group: nuodb
    spec:
      securityContext:
        runAsUser: 0
        fsGroup: 0
      containers:
      - name: sm
        image: nuodb/nuodb-ce:latest
        imagePullPolicy: IfNotPresent
        args: [ "nuosm", "--servers-ready-timeout", "300" ]
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - { name: DB_NAME, value: "nuodb"}
        - { name: DB_USER, value: "dba"}
        - { name: DB_PASSWORD, value: "goalie"}
        - { name: NUOCMD_API_SERVER,   value: "admin.testns.svc:8888"}
        - { name: PEER_ADDRESS,        value: "admin.testns.svc" }
        - { name: NUODB_OPTIONS,       value: "mem 1g" }
        ports:
        - containerPort: 48006
          protocol: TCP
        resources:
          requests:
            memory: "1Gi"
            cpu: "200m"
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - mountPath: /var/opt/nuodb/archive
          name: archive
  volumeClaimTemplates:
  - metadata:
      name: archive
      annotations:
        volume.beta.kubernetes.io/storage-class: openebs-cstor-sparse
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 5G
```

**nuodb-te.yaml**

```
---
apiVersion: apps/v1beta1
kind: Deployment
metadata:
  name: te
  labels:
    app: nuodb
    group: nuodb
    deploymentconfig: te
spec:
  replicas: 1
  template:
    metadata:
      labels:
        app: te
        deploymentconfig: te
        database: nuodb
        nodetype: te
    spec:
      containers:
      -
        name: te
        image: nuodb/nuodb-ce:latest
        imagePullPolicy: IfNotPresent
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - { name: DB_NAME,             value: "nuodb" }
        - { name: NUOCMD_API_SERVER,   value: "admin.testns.svc:8888"}
        - { name: PEER_ADDRESS,        value: "admin.testns.svc" }
        - { name: NUODB_LABELS,        value: "" }
        - { name: NUODB_OPTIONS,       value: "mem 1g" }
        args: [ "nuote", "--servers-ready-timeout", "300", "--database-created-timeout", "300" ]
        ports:
        - containerPort: 48006
          protocol: TCP
        resources:
          requests:
            memory: "1Gi"
            cpu: "200m"
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
        volumeMounts:
        - name: logdir
          mountPath: /var/log/nuodb
      volumes:
        - name: logdir
          emptyDir: {}
```

<br>

<hr>

<br>





## See Also:

<br>

<hr>

<br>

### [OpenEBS architecture](/docs/next/architecture.html)

### [OpenEBS use cases](/docs/next/usecases.html)

### [cStor pools overview](/docs/next/cstor.html#cstor-pools)



<br>

<hr>

<br>

