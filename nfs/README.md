# Dynamic NFS Provisioning in Kubernetes

##### Table of Contents  
* [1. Install NFS Server](#1-Install-NFS-Server)
	* [1.1 Create a folder which will be exported via NFS](#11-Create-a-folder-which-will-be-exported-via-NFS)
	* [1.2 Install the server](#12-Install-the-server-on-a-master-node)
	* [1.3 Check a status and start NFS](#13-Check-a-status-and-start-NFS)
	* [1.4 Check if NFS is running on the expected `IP_ADDRESS`](#14-Run-the-command-to-check-if-NFS-is-running-on-the-expected-IP_ADDRESS)
	* [1.5 Edit the exports file to add the file system we created to be exported to remote hosts](#15-Edit-the-exports-file-to-add-the-file-system-we-created-to-be-exported-to-remote-hosts)
	* [1.6 Make the local directory available to remote hosts](#16-Run-the-exportfs-command-to-make-the-local-directory-we-configured-available-to-remote-hosts)
* [2. Test the NFS configuration](#2-Test-the-NFS-configuration)
* [3. Install NFS client-provisioner](#3-Install-NFS-client-provisioner)
	* [3.1 Install with Helm chart](#31-Install-with-Helm-chart)
	* [3.2 Manual step-by-step installation](#32-Manual-step-by-step-installation)
* [4. Test the client: create PV, PVC and POD](#4-Test-the-client-create-PV-PVC-and-POD)
* [5. Delete POD,PV,PVC and other resources created for the client](#5-Delete-PODPVPVC-and-other-resources-created-for-the-client)
* [Resources](#Resources)


### 1. Install NFS Server

#### 1.1 Create a folder which will be exported via NFS

```
sudo mkdir /srv/nfs/kubedata -p
```
#### 1.2 Install the server (on a `master` node):
```
sudo apt install nfs-kernel-server
```


#### 1.3 Check a status and start NFS:
```
sudo service nfs-kernel-server status
```
User stop/start/restart commands

#### 1.4 Run the command to check if NFS is running on the expected `IP_ADDRESS`

```
nc -zvw3 192.168.56.2 2049
```
(As soon as NFS has been installed on a master node, the `IP_ADDRESS` is a master node IP. )

#### 1.5 Edit the exports file to add the file system we created to be exported to remote hosts.
```
sudo vi /etc/exports
```
Add the configuration:
```
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

#### 1.6 Run the `exportfs` command to make the local directory we configured available to remote hosts.

```
sudo exportfs -rav
```
Output:
```
exporting *:/srv/nfs/kubedata
```

If you want to see more details about our export file system, you can run “exportfs -v”.

```
sudo exportfs -v
```
Output:
```
/srv/nfs/kubedata
		<world>(rw,wdelay,insecure,no_root_squash,no_subtree_check,sec=sys,rw,insecure,no_root_squash,no_all_squash)
```

### 2. Test the NFS configuration
2.1 Log onto one of the worker nodes, mount the nfs filesystem and verify it:

```
vagrant@kubenode01:~$ 
sudo mount -t nfs 192.168.56.2:/srv/nfs/kubedata /mnt
```

Here you may have an [issue](https://askubuntu.com/questions/525243/why-do-i-get-wrong-fs-type-bad-option-bad-superblock-error)  with mounting. 

To fix the issue you need to install `nfs-common` on every node (master and workers):

```
vagrant@kubenode01:~$ 
sudo apt install nfs-common
```

Check the mount:
```
vagrant@kubenode01:~$ mount | grep kubedata
192.168.56.2:/srv/nfs/kubedata on /mnt type nfs4 (rw,relatime,vers=4.2,rsize=524288,wsize=524288,namlen=255,hard,proto=tcp,timeo=600,retrans=2,sec=sys,clientaddr=192.168.56.3,local_lock=none,addr=192.168.56.2)
```

2.2 Check the shared folder. As an example, create a file `a.out` in `/srv/nfs/kubedata`:

```
vagrant@kubemaster:/srv/nfs/kubedata 
$ touch a.out
vagrant@kubemaster:/srv/nfs/kubedata 
$ ls -l
total 8
drwxrwxrwx 2 root    root    4096 Mar 14 09:32 ./
drwxrwxrwx 3 root    root    4096 Mar 13 19:51 ../
-rw-rw-r-- 1 vagrant vagrant    0 Mar 14 09:32 a.out

```
This file should be now accessible on the `worker`node:
```
vagrant@kubenode01:~$ ls -l /mnt/
total 0
-rw-rw-r-- 1 vagrant vagrant 0 Mar 14 09:32 a.out
```

2.3 After verifying that NFS is configured correctly and working we can unmount the filesystem. Run the command on the `worker` node:

```
sudo umount /mnt
```
### 3. Install NFS client-provisioner

The next step is to install `NFS client-provisioner`. There are a few options how to do it:
* Install with a Helm-chart (run just one command)
* Install manually step-by-step

Let's consider both options

#### 3.1 Install with Helm chart

3.1.1 Run the command to install a client-provisioner:
```
helm install --set nfs.server=192.168.56.2 --set nfs.path=/srv/nfs/kubedata/ ckotzbauer/nfs-client-provisioner --generate-name
```

Here we specify two parameters:
* `nfs.server=192.168.56.2` - IP address of the NFS server
* `nfs.path=/srv/nfs/kubedata/` - a path which has been exported via NFS

3.1.2 Check the installation

Check the chart and installed pod:

```
vagrant@kubemaster:~ 
$ helm ls
NAME                             	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART                       	APP VERSION
nfs-client-provisioner-1615715679	default  	1       	2021-03-14 09:54:41.495997762 +0000 UTC	deployed	nfs-client-provisioner-1.0.2	3.1.0

vagrant@kubemaster:~ 
$ kubectl get po
NAME                                                 READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-1615715679-5b9fb655db-p5jcb   1/1     Running   0          12m
```

Check a Service Account and Role Bindings:
```
vagrant@kubemaster:~ 
$ kubectl get clusterrole,clusterrolebinding,role,rolebinding | grep nfs
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-1615715679-runner                               2021-03-14T09:54:41Z
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner-1615715679                  ClusterRole/nfs-client-provisioner-1615715679-runner                               15m
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner-1615715679   2021-03-14T09:54:41Z
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner-1615715679   Role/leader-locking-nfs-client-provisioner-1615715679   15m

```

Check a storageclass:
```
vagrant@kubemaster:~ 
$ kubectl get storageclass
NAME         PROVISIONER                                       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
nfs-client   cluster.local/nfs-client-provisioner-1615715679   Delete          Immediate           true                   17m
```

3.1.3 Create PV, PVC and POD to test the client

To test the deployment just run the steps described in 4.1-4.2 In 4.1 you need to use `helm-pvc-nfs.yaml` instead of `4-pvc-nfs.yaml` to create PVC and PV.

3.1.4 Uninstall a client-provisioner

```
vagrant@kubemaster:~ 
$ helm uninstall nfs-client-provisioner-1615715679
release "nfs-client-provisioner-1615715679" uninstalled
```

#### 3.2 Manual step-by-step installation

You can check out detail instructions in [How‌ to‌ Setup‌ Dynamic‌ NFS‌ Provisioning‌ Server‌ For‌ Kubernetes](https://redblink.com/setup-nfs-server-provisioner-kubernetes/). (Look into the section "Dynamic NFS Provisioning in Kubernetes").

Here is a short summary

3.2.1 Deploying Service Account and Role Bindings

```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl create -f rbac.yaml
serviceaccount/nfs-client-provisioner created
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner created
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner created
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner created
```
Verify that the service account, clusterrole and binding was created:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl get clusterrole,clusterrolebinding,role,rolebinding | grep nfs
clusterrole.rbac.authorization.k8s.io/nfs-client-provisioner-runner                                          2021-03-14T10:30:41Z
clusterrolebinding.rbac.authorization.k8s.io/run-nfs-client-provisioner                             ClusterRole/nfs-client-provisioner-runner                                          59s
role.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner   2021-03-14T10:30:41Z
rolebinding.rbac.authorization.k8s.io/leader-locking-nfs-client-provisioner   Role/leader-locking-nfs-client-provisioner   59s
```

3.2.2  Deploy Storage Class
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl create -f class.yaml
storageclass.storage.k8s.io/managed-nfs-storage created
```

Verify that the storage class was created:
```
$ kubectl get storageclass
NAME                  PROVISIONER       RECLAIMPOLICY   VOLUMEBINDINGMODE   ALLOWVOLUMEEXPANSION   AGE
managed-nfs-storage   example.com/nfs   Delete          Immediate           false                  51s
```

3.3.3 Deploying NFS client-provisioner
Before deploying make sure that correct values for IP_address of the NFS and path are specified in `deployment.yaml`. In our example we use:
* NFS IP-address: `192.168.56.2`
* Path: `/srv/nfs/kubedata`
```
      containers:
        - name: nfs-client-provisioner
          image: quay.io/external_storage/nfs-client-provisioner:latest
          volumeMounts:
            - name: nfs-client-root
              mountPath: /persistentvolumes
          env:
            - name: PROVISIONER_NAME
              value: example.com/nfs
            - name: NFS_SERVER
              value: 192.168.56.2
            - name: NFS_PATH
              value: /srv/nfs/kubedata
      volumes:
        - name: nfs-client-root
          nfs:
            server: 192.168.56.2
            path: /srv/nfs/kubedata
```
Deploy the client-provisioner:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl create -f deployment.yaml
deployment.apps/nfs-client-provisioner created
```
Verify the deployment:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl get all
NAME                                          READY   STATUS    RESTARTS   AGE
pod/nfs-client-provisioner-6b5c7f8f75-w4wfk   1/1     Running   0          42s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   203d

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-client-provisioner   1/1     1            1           42s

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-client-provisioner-6b5c7f8f75   1         1         1       42s
```
### 4. Test the client: create PV, PVC and POD
When client is installed we can perform some testing.

4.1 Creating Persistent Volume and Persistent Volume Claims

Check is there is any PV and PVC:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl get pv,pvc
No resources found in default namespace.
```
Also, we can look in the directory we allocated for Persistent Volumes:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ ls -l /srv/nfs/kubedata/
total 0
```
Create PVC (note, we don't create PV here):
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl create -f 4-pvc-nfs.yaml
persistentvolumeclaim/pvc1 created
```

Check is there is any PVC and **PV** have been created:
```
$ kubectl get pvc,pv
NAME                         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS          AGE
persistentvolumeclaim/pvc1   Bound    pvc-cb32cece-a85d-4e76-abd4-d14df4f685dd   500Mi      RWX            managed-nfs-storage   65s

NAME                                                        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS          REASON   AGE
persistentvolume/pvc-cb32cece-a85d-4e76-abd4-d14df4f685dd   500Mi      RWX            Delete           Bound    default/pvc1   managed-nfs-storage            64s
```
As we can see that both PVC and PV have been created. Note, that **PV** has *dynamically* been created!


Check the directory that we allocated for Persistent Volumes:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ ls -l /srv/nfs/kubedata/
total 4
drwxrwxrwx 2 root root 4096 Mar 14 10:53 default-pvc1-pvc-cb32cece-a85d-4e76-abd4-d14df4f685dd
```
We can see that a new folder has been created there.

4.2 Create a Pod to use Persistent Volume Claims

```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ kubectl create -f 4-busybox-pv-nfs.yaml 
pod/busybox created
```
Check the created pod:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k get po
NAME                                      READY   STATUS    RESTARTS   AGE
busybox                                   1/1     Running   0          21s
nfs-client-provisioner-6b5c7f8f75-w4wfk   1/1     Running   0          37m
```
Log onto the pod and create a test file `test.txt` in a shared path:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k exec -it busybox -- /bin/sh
/ # ls -l /mydata/
total 0
/ # touch /mydata/test.txt
/ # ls -l /mydata/
total 0
-rw-r--r--    1 root     root             0 Mar 14 11:22 test.txt
/ # 
```
Log out from the POD and check the directory allocated for Persistent Volumes:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ ls -l /srv/nfs/kubedata/default-pvc1-pvc-cb32cece-a85d-4e76-abd4-d14df4f685dd/
total 0
-rw-r--r-- 1 root root 0 Mar 14 11:22 test.txt
```
As we can see the file `test.txt` is shared across the master node and the pod.

### 5. Delete POD,PV,PVC and other resources created for the client
Now lets undeploy the created resources

Delete the `busybox` pod:

```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k delete -f 4-busybox-pv-nfs.yaml 
pod "busybox" deleted
```
Delete PVC:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k delete -f 4-pvc-nfs.yaml 
persistentvolumeclaim "pvc1" deleted
```
Verify that PVC and PV are deleted:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k get pv,pvc
No resources found in default namespace.
```
As we can see both PVC and dynamically created PV have been deleted.


Verify that PV has been also deleted on the shared path:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ ls -l /srv/nfs/kubedata/
total 0
```

Uninstall the client-provisioner:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k delete -f deployment.yaml 
deployment.apps "nfs-client-provisioner" deleted
```

Undeploy storageclass:
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k delete -f class.yaml 
storageclass.storage.k8s.io "managed-nfs-storage" deleted
```

Undeploy Service Account and Role Bindings
```
vagrant@kubemaster:~/samples/k8s-samples/nfs (main)
$ k delete -f rbac.yaml 
serviceaccount "nfs-client-provisioner" deleted
clusterrole.rbac.authorization.k8s.io "nfs-client-provisioner-runner" deleted
clusterrolebinding.rbac.authorization.k8s.io "run-nfs-client-provisioner" deleted
role.rbac.authorization.k8s.io "leader-locking-nfs-client-provisioner" deleted
rolebinding.rbac.authorization.k8s.io "leader-locking-nfs-client-provisioner" deleted
```

### Resources
* [How‌ ‌to‌ ‌Setup‌ Dynamic‌ ‌NFS‌ ‌Provisioning‌ ‌Server‌ ‌For‌ ‌Kubernetes?‌](https://redblink.com/setup-nfs-server-provisioner-kubernetes/)
* Kubeapps hub: [nfs-client-provisioner](https://hub.kubeapps.com/charts/ckotzbauer/nfs-client-provisioner)
* Artifact hub: [nfs-client-provisioner](https://artifacthub.io/packages/helm/ckotzbauer/nfs-client-provisioner)


