under constuction
# Elastic Cloud on Kubernetes

Here is a short summary on how to deploy ECK

### 1.Install NFS client-provisioner
For details check out [Dynamic NFS Provisioning in Kubernetes](../nfs#dynamic-nfs-provisioning-in-kubernetes)

When NFS server is up and running just run the command:
```
helm install --set nfs.server=192.168.56.2 --set nfs.path=/srv/nfs/kubedata/ ckotzbauer/nfs-client-provisioner --generate-name
```

### 2.Install ECK

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-quickstart.html

#### 2.1 Deploy ECK in your Kubernetes cluster

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-eck.html

##### 2.1.1 Install custom resource definitions and the operator with its RBAC rules

```
kubectl apply -f https://download.elastic.co/downloads/eck/1.4.1/all-in-one.yaml
```

##### 2.1.2 Monitor the operator logs:
```
kubectl -n elastic-system logs -f statefulset.apps/elastic-operator
```

#### 2.2 Deploy an Elasticsearch cluster
https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html

```
cat <<EOF | kubectl apply -f -
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: quickstart
spec:
  version: 7.11.2
  nodeSets:
  - name: default
    count: 1
    config:
      node.store.allow_mmap: false
EOF
```

Check a deployemnt.
After this step you may see that a pod `quickstart-es-default-0` is in a pending state:
```
$ k get po
NAME                                                READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-1616266763-db4bd4646-25kmw   1/1     Running   1          15h
quickstart-es-default-0                             0/1     Pending   0          4m16s
```

This happens because a created PVC doesn't specify a correct `storageclass`. If you check a created PVC is will also have a pending status:
```
$ k get pvc
NAME                                         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-quickstart-es-default-0   Pending                                                     7m14s
```
(Question: how to provide a custom `storageclass` when deploying a custom resource `Elasticsearch`???)

How to fix this:


1. Get specification for a created PVC and store it as a yaml file:

```
kubectl get pvc elasticsearch-data-quickstart-es-default-0 -o yaml > pvc.yaml
```

2. Modify the `pvc.yaml`: replace the section `spec:` whit a new one:
```
  spec:
    storageClassName: nfs-client
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

3. Replace PVC: delete the current one and create a new one:
```
kubectl delete pvc elasticsearch-data-quickstart-es-default-0
kubectl create -f pvc.yaml
```

4. Check a new PVC and created PV:


``` 
$ k get pvc,pv
NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/elasticsearch-data-quickstart-es-default-0   Bound    pvc-da94c602-3bf9-40d8-acfd-cf8a12b6e7e0   1Gi        RWO            nfs-client     5m9s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                STORAGECLASS   REASON   AGE
persistentvolume/pvc-da94c602-3bf9-40d8-acfd-cf8a12b6e7e0   1Gi        RWO            Delete           Bound    default/elasticsearch-data-quickstart-es-default-0   nfs-client              5m9s
```


Check pods:
```
$ k get po
NAME                                                READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-1616266763-db4bd4646-25kmw   1/1     Running   1          16h
quickstart-es-default-0                             1/1     Running   0          15m
```

#### 2.3 Deploy a Kibana instance

https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-kibana.html#k8s-deploy-kibana

```
cat <<EOF | kubectl apply -f -
apiVersion: kibana.k8s.elastic.co/v1
kind: Kibana
metadata:
  name: quickstart
spec:
  version: 7.11.2
  count: 1
  elasticsearchRef:
    name: quickstart
EOF
```
