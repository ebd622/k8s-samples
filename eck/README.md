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


