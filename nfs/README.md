# Dynamic NFS Provisioning in Kubernetes


### Install NFS Server

1. Create a folder which will be exported via NFS

```
sudo mkdir /srv/nfs/kubedata -p
```
2. Install the server (on a `master` node):
```
sudo apt install nfs-kernel-server
```


3. Check a status and start NFS:
```
sudo service nfs-kernel-server status
```
User stop/start/restart commands

4. Run the command to check if NFS is running on the expected `IP_ADDRESS`

```
nc -zvw3 192.168.56.2 2049
```
(As soon as NFS has been installed on a master node, the `IP_ADDRESS` is a master node IP. )

5. Edit the exports file to add the file system we created to be exported to remote hosts.
```
sudo vi /etc/exports

### Add the configuration:
/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash,no_all_squash,insecure)
```

6. Run the `exportfs` command to make the local directory we configured available to remote hosts.

```
sudo exportfs -rav

### This will print:
exporting *:/srv/nfs/kubedata
```

If you want to see more details about our export file system, you can run “exportfs -v”.

```
sudo exportfs -v
```
