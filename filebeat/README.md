# Running Filebeat on Kubernetes

Here is  a step-by-step instruction on how to set up and run [Filebeat on Keubernetes](https://www.elastic.co/guide/en/beats/filebeat/6.8/running-on-kubernetes.html).


### 1. Download a filebeat manifest

```
curl -L -O https://raw.githubusercontent.com/elastic/beats/6.8/deploy/kubernetes/filebeat-kubernetes.yaml
```
The manifest needs to be modified before deploying to a cluster.


### 2. Make changes in the manifest

#### 2.1 Change a namespace

Filebeat should be deployed on the same namespace as Elastic and Kibana. In our example both Elastic and Kibana are deployed in the `default` namespace. 
In the manifest you need to replace `kube-system` with `default`

#### 2.2 Change a tag image
We agre going to use the image tag `7.5.2`:
```
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.8.14
```

#### 2.3 Add certificate to connect to Elastic
Elastic is availbale via https, here we need to add a certificate to allow Filebeat to connect to Elastic via https:
```
      ssl.certificate_authorities:
        - /etc/certificates/ca.crt 
```
This certificate needs to be available in the filebeat container.


#### 2.4 Add a voulme
Add a new volume into the `volumes` section:
```
     - name: certs
       secret:
         secretName: quickstart-es-http-certs-public
```
(The name `certs` is used here but any other name is also possible)

The added secret `quickstart-es-http-certs-public` is already available on a cluster, it has been deployed with Elastic:

```
$ kubectl get secrets quickstart-es-http-certs-public
NAME                              TYPE     DATA   AGE
quickstart-es-http-certs-public   Opaque   2      130m
```

5. Mount created vlume:

```
     - name: certs
       mountPath: /etc/certificates/ca.crt
       readOnly: true
       subPath: ca.crt
```

6. Other changes in Deamonset

```
$ kubectl get svc
NAME                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)          AGE
kubernetes                ClusterIP   10.96.0.1        <none>        443/TCP          211d
quickstart-es-default     ClusterIP   None             <none>        9200/TCP         136m
quickstart-es-http        ClusterIP   10.103.76.57     <none>        9200/TCP         136m
quickstart-es-transport   ClusterIP   None             <none>        9300/TCP         136m
quickstart-kb-http        NodePort    10.105.114.130   <none>        5601:31025/TCP   118m
```

```
$ echo $PASSWORD
xNV99jXC40IUnwF8l2h7W299
```

Change `ELASTICSEARCH_HOST` and `ELASTICSEARCH_PASSWORD`:

```
    - name: ELASTICSEARCH_HOST
      value: https://quickstart-es-http
    - name: ELASTICSEARCH_PASSWORD
      value: changeme

```
