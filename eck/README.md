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

