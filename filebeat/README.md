# Running Filebeat on Kubernetes

Here is  a step-by-step instruction how to set up and run [Filebeat on Keubernetes](https://www.elastic.co/guide/en/beats/filebeat/6.8/running-on-kubernetes.html).


### 1. Download a filebeat manifest

```
curl -L -O https://raw.githubusercontent.com/elastic/beats/6.8/deploy/kubernetes/filebeat-kubernetes.yaml
```



### 2. Make changes in the manifest

#### 2.1 Change Namespace

Filebeat needs to be deployed on the same namespace as Elastic and Kibana. Replace `kube-system` with `default`

2. Change image?
Use the image 7.5.2:
```
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:6.8.14
```

3. Add certificate to connect to Elastic

```
      ssl.certificate_authorities:
        - /etc/certificates/ca.crt 
```
This certificate needs to be available in the filebeat container


4. Add one more voulme:

```
     - name: certs
       secret:
         secretName: quickstart-es-http-certs-public
```
The secret `quickstart-es-http-certs-public` is one deployed with Elastic:

```
$ k get secrets quickstart-es-http-certs-public
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
