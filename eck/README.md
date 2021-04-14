under constuction
# Elastic Cloud on Kubernetes

## Table of Contents
* [1. Install NFS client-provisioner](#1-Install-NFS-client-provisioner)
* [2. Install ECK](#2-Install-ECK)
  * [2.1 Deploy ECK in your Kubernetes cluster]()
  * [2.2 Deploy an Elasticsearch cluster]()
  * [2.3 Test Elasticsearch]()
  * [2.4 Deploy a Kibana instance]()
  * [2.5 Check a deployed elastic stack (elasticsearch and kibana)](#25-check-a-deployed-elastic-stack-elasticsearch-and-kibana)
* [3. Uninstall ECK and Kibana](#3-Uninstall-ECK-and-Kibana)
  * [3.1 Delete ECK and Kibana deployments](#31-delete-eck-and-kibana-deployments)
  * [3.2 Uninstall custom resource definitions and the operator with its RBAC rules](#32-uninstall-custom-resource-definitions-and-the-operator-with-its-rbac-rules)
  * [3.3 Uninstall NFS client-provisioner](#33-uninstall-nfs-client-provisioner)

---
Here is a short summary on how to deploy ECK

### 1. Install NFS client-provisioner
For details check out [Dynamic NFS Provisioning in Kubernetes](../nfs#dynamic-nfs-provisioning-in-kubernetes)

When NFS server is up and running just run the command:
```
helm install --set nfs.server=192.168.56.2 --set nfs.path=/srv/nfs/kubedata/ ckotzbauer/nfs-client-provisioner --generate-name
```

### 2. Install ECK

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
$ kubectl get po
NAME                                                READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-1616266763-db4bd4646-25kmw   1/1     Running   1          15h
quickstart-es-default-0                             0/1     Pending   0          4m16s
```

This happens because a created PVC doesn't specify a correct `storageclass`. If you check a created PVC is will also have a pending status:
```
$ kubectl get pvc
NAME                                         STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
elasticsearch-data-quickstart-es-default-0   Pending                                                     7m14s
```
(Question: how to provide a custom `storageclass` when deploying a custom resource `Elasticsearch`???)

How to fix this:


##### 2.2.1 Get specification for a created PVC and store it as a yaml file:

```
kubectl get pvc elasticsearch-data-quickstart-es-default-0 -o yaml > pvc.yaml
```

##### 2.2.2 Modify the `pvc.yaml`: replace the section `spec:` whit a new one:
```
  spec:
    storageClassName: nfs-client
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

##### 2.2.3 Replace PVC: delete the current one and create a new one:
```
kubectl delete pvc elasticsearch-data-quickstart-es-default-0
kubectl create -f pvc.yaml
```

##### 2.2.4 Check a new PVC and created PV:


``` 
$ kubectl get pvc,pv
NAME                                                               STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/elasticsearch-data-quickstart-es-default-0   Bound    pvc-da94c602-3bf9-40d8-acfd-cf8a12b6e7e0   1Gi        RWO            nfs-client     5m9s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                                                STORAGECLASS   REASON   AGE
persistentvolume/pvc-da94c602-3bf9-40d8-acfd-cf8a12b6e7e0   1Gi        RWO            Delete           Bound    default/elasticsearch-data-quickstart-es-default-0   nfs-client              5m9s
```


Check pods:
```
$ kubectl get po
NAME                                                READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-1616266763-db4bd4646-25kmw   1/1     Running   1          16h
quickstart-es-default-0                             1/1     Running   0          15m
```

#### 2.3 Test Elasticsearch

When Elasticsearch is up an running you can [Requset Elasticsearch access](https://www.elastic.co/guide/en/cloud-on-k8s/current/k8s-deploy-elasticsearch.html#k8s_request_elasticsearch_access) and perofm testing.

##### 2.3.1 Get credentials

A default user named `elastic` is automatically created with the password stored in a Kubernetes secret. Run the command to get a password and store it in the variable `PASSWORD`:
```
PASSWORD=$(kubectl get secret quickstart-es-elastic-user -o go-template='{{.data.elastic | base64decode}}')
```

##### 2.3.2 Expose the service `quickstart-es-http` out of a cluster. <br/>
By default the service is available within a cluster, it uses `ClusterPort`. You need to edit the service and replace `ClusterPort` with `NodePort`. Then you will get a port to access Elastic outside a cluster:

```
$ kubectl get svc quickstart-es-http
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
quickstart-es-http   NodePort   10.96.104.110   <none>        9200:30449/TCP   47m
```

##### 2.3.3 Access Elastic outside a cluster

(remember that the env-variable `PASSWORD`may be defined in a different terminal)
```
$ curl -u "elastic:$PASSWORD" -k "https://kubemaster:30449"
{
  "name" : "quickstart-es-default-0",
  "cluster_name" : "quickstart",
  "cluster_uuid" : "ept3QhxkR4mM5thyb-ZxQQ",
  "version" : {
    "number" : "7.11.2",
    "build_flavor" : "default",
    "build_type" : "docker",
    "build_hash" : "3e5a16cfec50876d20ea77b075070932c6464c7d",
    "build_date" : "2021-03-06T05:54:38.141101Z",
    "build_snapshot" : false,
    "lucene_version" : "8.7.0",
    "minimum_wire_compatibility_version" : "6.8.0",
    "minimum_index_compatibility_version" : "6.0.0-beta1"
  },
  "tagline" : "You Know, for Search"
}
```

#### 2.4 Deploy a Kibana instance

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

To expose Kibana UI out a cluster you need to modify the service `quickstart-kb-http` and replace the port `ClusterIP` with `NodePort`.

Check the service and an external port:

```
$ k get svc quickstart-kb-http
NAME                 TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
quickstart-kb-http   NodePort   10.105.114.130   <none>        5601:31025/TCP   5m10s
```

#### 2.5 Check a deployed elastic stack (elasticsearch and kibana)

```
$ kubectl get elastic
NAME                                                    HEALTH   NODES   VERSION   PHASE   AGE
elasticsearch.elasticsearch.k8s.elastic.co/quickstart   green    1       7.11.2    Ready   8h

NAME                                      HEALTH   NODES   VERSION   AGE
kibana.kibana.k8s.elastic.co/quickstart   green    1       7.11.2    8h

```

### 3. Uninstall ECK and Kibana

#### 3.1 Delete ECK and Kibana deployments
```
kubectl delete kibana quickstart
kubectl delete elasticsearch quickstart
```

#### 3.2 Uninstall custom resource definitions and the operator with its RBAC rules

```
kubectl delete -f https://download.elastic.co/downloads/eck/1.4.1/all-in-one.yaml
```

#### 3.3 Uninstall NFS client-provisioner
```
helm uninstall nfs-client-provisioner-xxxxxxxxxx
```
