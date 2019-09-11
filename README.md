## RocketMQ Operator
[![License](https://img.shields.io/badge/license-Apache%202-4EB1BA.svg)](https://www.apache.org/licenses/LICENSE-2.0.html)

## Overview

RocketMQ Operator is to manage RocketMQ service instances deployed on the Kubernetes cluster.
It is built using the [Operator SDK](https://github.com/operator-framework/operator-sdk), which is part of the [Operator Framework](https://github.com/operator-framework/).

## Quick Start

### Prepare Storage Class

The first step is to prepare a storage class to create PV and PVC where the RocketMQ data will be stored. Here we use NFS as the storage class.

1. Deploy NFS server and clients on your Kubernetes cluster. Please make sure they are functional before you go to the next step.

2. Modify the following configurations of the ```deploy/storage/nfs-client.yaml``` file:
``` 
...
            - name: NFS_SERVER
              value: 192.168.130.32
            - name: NFS_PATH
              value: /data/k8s
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.130.32
            path: /data/k8s
...
```
Replace ```192.168.130.32``` and ```/data/k8s``` with your true NFS server IP address and NFS server data volume path.
 
3. Create a NFS storage class for RocketMQ, run

```
$ cd deploy/storage
$ ./deploy-storage-class.sh
```
If the storage class is successfully deployed, you can get the pod status like:
```
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-7cf858f754-7vxmm   1/1     Running   0          136m
```

### Define Your RocketMQ Cluster

RocketMQ Operator provides several CRDs to allow users define their RocketMQ service component cluster, which includes the Namesrv cluster and the Broker cluster.

1. Check the file ```rocketmq_v1alpha1_nameservice_cr.yaml``` in the ```deploy/crds``` directory, for example:
```
apiVersion: rocketmq.apache.org/v1alpha1
kind: NameService
metadata:
  name: name-service
spec:
  # size is the the name service instance number of the name service cluster
  size: 1
  # nameServiceImage is the customized docker image repo of the RocketMQ name service
  nameServiceImage: docker.io/library/rocketmq-namesrv:4.5.0-alpine
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: namesrv-storage
        annotations:
          volume.beta.kubernetes.io/storage-class: rocketmq-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```

which defines the RocketMQ name service (namesrv) cluster scale.

2. Check the file ```cache_v1alpha1_broker_cr.yaml``` in the ```deploy/crds``` directory, for example:
```
apiVersion: rocketmq.apache.org/v1alpha1
kind: Broker
metadata:
  name: broker
spec:
  # size is the number of the broker cluster, each broker cluster contains a master broker and [replicaPerGroup] replica brokers.
  size: 2
  # nameServers is the [ip:port] list of name service
  nameServers: 192.168.130.33:9876
  # replicationMode is the broker replica sync mode, can be ASYNC or SYNC
  replicationMode: ASYNC
  # replicaPerGroup is the number of each broker cluster
  replicaPerGroup: 1
  # brokerImage is the customized docker image repo of the RocketMQ broker
  brokerImage: docker.io/library/rocketmq-broker:4.5.0-alpine
  # imagePullPolicy is the image pull policy
  imagePullPolicy: Always
  # allowRestart defines whether allow pod restart
  allowRestart: false
  # volumeClaimTemplates defines the storageClass
  volumeClaimTemplates:
    - metadata:
        name: broker-storage
        annotations:
          volume.beta.kubernetes.io/storage-class: rocketmq-storage
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 8Gi
``` 
which defines the RocketMQ broker cluster scale, the [ip:port] list of name service and so on.

### Deploy RocketMQ Operator & Create RocketMQ Cluster

1. To deploy the RocketMQ Operator on your Kubernetes cluster, please run the following script:

```
$ ./install-operator.sh
```

Use command ```kubectl get pods``` to check the RocketMQ Operator deploy status like:

```
$ kubectl get pods
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-7cf858f754-7vxmm   1/1     Running   0          146m
rocketmq-operator-564b5d75d-jllzk         1/1     Running   0          108s
```
Now you can use the CRDs provide by RocketMQ Operator to deploy your RocketMQ cluster.
 
2. Deploy the RocketMQ name service cluster by running:

``` 
$ kubectl apply -f deploy/crds/rocketmq_v1alpha1_nameservice_cr.yaml 
nameservice.rocketmq.apache.org/name-service created
```

Check the status:

```
$ kubectl get pods -owide
NAME                                      READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
name-service-0                            1/1     Running   0          3m18s   192.168.130.33   k2data-13   <none>           <none>
nfs-client-provisioner-7cf858f754-7vxmm   1/1     Running   0          150m    10.244.2.114     k2data-14   <none>           <none>
rocketmq-operator-564b5d75d-jllzk         1/1     Running   0          5m53s   10.244.2.116     k2data-14   <none>           <none>
```

We can see that there are 1 name service Pods running on 1 nodes and their IP addresses. Modify the ```nameServers``` field in the ```cache_v1alpha1_broker_cr.yaml``` file using the IP addresses.

3. Deploy the RocketMQ broker clusters by running:
```
$ kubectl apply -f deploy/crds/cache_v1alpha1_broker_cr.yaml 
broker.cache.example.com/broker created 
```

After a while after the Containers are created, the Kubernetes clusters status should be like:

``` 
$ kubectl get pods -owide
NAME                                      READY   STATUS    RESTARTS   AGE     IP               NODE        NOMINATED NODE   READINESS GATES
broker-0-master-0                         1/1     Running   0          38s     10.244.4.18      k2data-11   <none>           <none>
broker-0-replica-1-0                        1/1     Running   0          38s     10.244.1.128     k2data-13   <none>           <none>
broker-1-master-0                         1/1     Running   0          38s     10.244.2.117     k2data-14   <none>           <none>
broker-1-replica-1-0                        1/1     Running   0          38s     10.244.3.17      k2data-15   <none>           <none>
name-service-0                            1/1     Running   0          6m7s    192.168.130.33   k2data-13   <none>           <none>
nfs-client-provisioner-7cf858f754-7vxmm   1/1     Running   0          153m    10.244.2.114     k2data-14   <none>           <none>
rocketmq-operator-564b5d75d-jllzk         1/1     Running   0          8m42s   10.244.2.116     k2data-14   <none>           <none>
```

Check the PV and PVC status:
```
$ kubectl get pvc
NAME                                STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS       AGE
broker-storage-broker-0-master-0    Bound    pvc-7a74871b-c005-441a-bb15-8106566c9d19   8Gi        RWO            rocketmq-storage   78s
broker-storage-broker-0-replica-1-0   Bound    pvc-521e7e9a-3795-487a-9f76-22da74db74dd   8Gi        RWO            rocketmq-storage   78s
broker-storage-broker-1-master-0    Bound    pvc-d7b76efe-384c-4f8d-9e8a-ebe209ba826c   8Gi        RWO            rocketmq-storage   78s
broker-storage-broker-1-replica-1-0   Bound    pvc-af266db9-83a9-4929-a2fe-e40fb5fdbfa4   8Gi        RWO            rocketmq-storage   78s
namesrv-storage-name-service-0      Bound    pvc-c708cb49-aa52-4992-8cac-f46a48e2cc2e   1Gi        RWO            rocketmq-storage   79s
$ kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                       STORAGECLASS       REASON   AGE
pvc-521e7e9a-3795-487a-9f76-22da74db74dd   8Gi        RWO            Delete           Bound    default/broker-storage-broker-0-replica-1-0   rocketmq-storage            79s
pvc-7a74871b-c005-441a-bb15-8106566c9d19   8Gi        RWO            Delete           Bound    default/broker-storage-broker-0-master-0    rocketmq-storage            79s
pvc-af266db9-83a9-4929-a2fe-e40fb5fdbfa4   8Gi        RWO            Delete           Bound    default/broker-storage-broker-1-replica-1-0   rocketmq-storage            78s
pvc-c708cb49-aa52-4992-8cac-f46a48e2cc2e   1Gi        RWO            Delete           Bound    default/namesrv-storage-name-service-0      rocketmq-storage            79s
pvc-d7b76efe-384c-4f8d-9e8a-ebe209ba826c   8Gi        RWO            Delete           Bound    default/broker-storage-broker-1-master-0    rocketmq-storage            78s
```

Congratulations! You have successfully deployed your RocketMQ cluster by RocketMQ Operator.

### Verify the Data Storage

Access the NFS server node of your cluster and verify whether the RocketMQ data is stored in your NFS data volume path:

```
$ cd /data/k8s/
$ ls
default-broker-storage-broker-0-master-0-pvc-7a74871b-c005-441a-bb15-8106566c9d19   default-broker-storage-broker-1-replica-1-0-pvc-af266db9-83a9-4929-a2fe-e40fb5fdbfa4
default-broker-storage-broker-0-replica-1-0-pvc-521e7e9a-3795-487a-9f76-22da74db74dd  default-namesrv-storage-name-service-0-pvc-c708cb49-aa52-4992-8cac-f46a48e2cc2e
default-broker-storage-broker-1-master-0-pvc-d7b76efe-384c-4f8d-9e8a-ebe209ba826c
$ ls default-broker-storage-broker-1-master-0-pvc-d7b76efe-384c-4f8d-9e8a-ebe209ba826c/logs/rocketmqlogs/
broker_default.log  broker.log  commercial.log  filter.log  lock.log  protection.log  remoting.log  stats.log  storeerror.log  store.log  transaction.log  watermark.log
$ cat default-broker-storage-broker-1-master-0-pvc-d7b76efe-384c-4f8d-9e8a-ebe209ba826c/logs/rocketmqlogs/broker.log 
...
2019-09-10 14:12:22 INFO main - The broker[broker-1-master-0, 10.244.2.117:10911] boot success. serializeType=JSON and name server is 192.168.130.33:9876
...
```




### Clean the Environment
If you want to tear down the RocketMQ cluster, to remove the broker clusters run

```
$ kubectl delete -f deploy/crds/cache_v1alpha1_broker_cr.yaml 
```

to remove the name service clusters:

```
$ kubectl delete -f deploy/crds/rocketmq_v1alpha1_nameservice_cr.yaml
```

to remove the RocketMQ Operator:

```
$ ./purge-operator.sh
```

to remove the storage class for RocketMQ:

```
$ cd deploy/storage
$ ./remove-storage-class.sh
```

> Note: the NFS persistence data will not be deleted by default.