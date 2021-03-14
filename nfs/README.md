# Dynamic NFS Provisioning in Kubernetes


### 1. Install NFS Server

1.1 Create a folder which will be exported via NFS

```
sudo mkdir /srv/nfs/kubedata -p
```
1.2 Install the server (on a `master` node):
```
sudo apt install nfs-kernel-server
```


1.3 Check a status and start NFS:
```
sudo service nfs-kernel-server status
```
User stop/start/restart commands

1.4 Run the command to check if NFS is running on the expected `IP_ADDRESS`

```
nc -zvw3 192.168.56.2 2049
```
(As soon as NFS has been installed on a master node, the `IP_ADDRESS` is a master node IP. )

1.5 Edit the exports file to add the file system we created to be exported to remote hosts.
```
sudo vi /etc/exports
```
Add the configuration:
```
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

1.6 Run the `exportfs` command to make the local directory we configured available to remote hosts.

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
