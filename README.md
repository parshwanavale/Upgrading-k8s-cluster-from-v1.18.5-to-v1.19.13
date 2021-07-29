# Upgrading-k8s-cluster-from-v1.18.5-to-v1.19.13
**There are basically two aspects :-**
+ Creating a backup for the existing version of cluster
+ Actual Upgrade of the cluster ( basically upgrading versions of each node)
---
## 1.Backup part
**Why backup Kubernetes?**

1. To be able to restore a failed control plane Node.
2. To be able to restore applications (with data).

**How to backup Kubernetes?**

1. Backup for the Etcd and relevant certificates in order to restore the control plane
2. Backup for the applications running in the cluster

**Steps:**
 
 First of all we need to copy the certificates which are present in /pki directory (You may not need to apply sudo as we are already loged in to root)
```
# Backup certificates in backup directory
sudo cp -r /etc/kubernetes/pki backup/
```
Taking snapshot on etcd
```
sudo docker run --rm -v $(pwd)/backup:/backup \
    --network host \
    -v /etc/kubernetes/pki/etcd:/etc/kubernetes/pki/etcd \
    --env ETCDCTL_API=3 \
    k8s.gcr.io/etcd:3.4.3-0 \
    etcdctl --endpoints=https://127.0.0.1:2379 \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
    --key=/etc/kubernetes/pki/etcd/healthcheck-client.key \
    snapshot save /backup/etcd-snapshot-latest.db
```

```
# Backup kubeadm-config
sudo cp /etc/kubeadm/kubeadm-config.yaml backup/
```
### The final command is optional and only relevant if you use a configuration file for kubeadm.
---
## Now when we want to restore the control plane we follow the exact opposite steps mentioned above
```
# Restore certificates
sudo cp -r backup/pki /etc/kubernetes/
```

```# Restore etcd backup
sudo mkdir -p /var/lib/etcd
sudo docker run --rm \
    -v $(pwd)/backup:/backup \
    -v /var/lib/etcd:/var/lib/etcd \
    --env ETCDCTL_API=3 \
    k8s.gcr.io/etcd:3.4.3-0 \
    /bin/sh -c "etcdctl snapshot restore '/backup/etcd-snapshot-latest.db' ; mv /default.etcd/member/ /var/lib/etcd/"
```

```
# Restore kubeadm-config
sudo mkdir /etc/kubeadm
sudo cp backup/kubeadm-config.yaml /etc/kubeadm/
```

```
# Initialize the master with backup
sudo kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd \
    --config /etc/kubeadm/kubeadm-config.yaml
```

___
---
## 2.The Upgrade Part
Before the backup process there are certain limitations we need to consider
+ You only can upgrade from one MINOR version to the next MINOR version, or between PATCH versions of the same MINOR. That is, you cannot skip MINOR versions when you upgrade. For example, you can upgrade from 1.y to 1.y+1, but not from 1.y to 1.y+2.
+ It is quiet likely the User may have a multimaster multiworker cluster, and we need to follow the same steps on each node separately (master specific and worker specific)
so one may think of using broadcast, however it is a good practice we do not use broadcast as it will help us acheving the `Zero Downtime.`


### Before you begin
+ You need to have a kubeadm Kubernetes cluster running version 1.18.0 or later.
+ Swap must be disabled.
+ The cluster should use a static control plane and etcd pods or external etcd.
+ Make sure you read the release notes carefully.
+ Make sure to back up any important components, such as app-level state stored in a database. kubeadm upgrade does not touch your workloads, only components internal to Kubernetes, but backups are always a best practice.

### Determine which version to upgrade to
```
yum list --showduplicates kubeadm --disableexcludes=kubernetes
# find the latest 1.19 version in the list
# it should look like 1.19.x-0, where x is the latest patch
```

### Upgrading control plane nodes [ Master ]
+ On your first control plane node, upgrade kubeadm:
```
# replace x in 1.19.x-0 with the latest patch version
yum install -y kubeadm-1.19.x-0 --disableexcludes=kubernetes
```
+ Verify that the download works and has the expected version:
```
kubeadm version
```
+ On the control plane node(master node), run:
```
sudo kubeadm upgrade plan
```
The output will show the current version and the available version of kubelet

+ Choose a version to upgrade to, and run the appropriate command. For example:
```
# replace x with the patch version you picked for this upgrade
sudo kubeadm upgrade apply v1.19.x
```
During the process you will see this message 
```
Are you sure you want to proceed with the upgrade?
```
Type yes and proceed

At the end of process you should see output similar to this:
```
[upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.19.0". Enjoy!

[upgrade/kubelet] Now that your control plane is upgraded, please proceed with upgrading your kubelets if you haven't already done so.
```
### Upgrade additional control plane nodes: 
Now if you have more than one control plane nodes [master1, master2...so on] you don't need to run all the commands mentioned above
as you have performed them on one of CPN(master) already,on the remaining CNIs you can just run
```
sudo kubeadm upgrade node
```
Also `sudo kubeadm upgrade plan` is also not needed

Now the next steps are to be followed on every CNI

### Drain the control plane node
+ Prepare the node for maintenance by marking it unschedulable and evicting the workloads:
```
# replace <cp-node-name> with the name of your control plane node (master_node)
kubectl drain <cp-node-name> --ignore-daemonsets
```

### Upgrade kubelet and kubectl
+ Upgrade the kubelet and kubectl on all control plane nodes:
```# replace x in 1.19.x-0 with the latest patch version
yum install -y kubelet-1.19.x-0 kubectl-1.19.x-0 --disableexcludes=kubernetes
```
+ Restart the kubelet
```
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

### Uncordon the control plane node
+ Bring the node back online by marking it schedulable:
```
# replace <cp-node-name> with the name of your control plane node (master)
kubectl uncordon <cp-node-name>
```
---
## Upgrade worker nodes

### Upgrade kubeadm
+ Upgrade kubeadm on all worker nodes:
```
# replace x in 1.19.x-0 with the latest patch version
yum install -y kubeadm-1.19.x-0 --disableexcludes=kubernetes
```

### Upgrade the kubelet configuration 
+ Call the following command:
```
sudo kubeadm upgrade node
```

### Drain the node
+ Prepare the node for maintenance by marking it unschedulable and evicting the workloads:
+ This can be performed by hitting following command on any of the CPN (master)
```
# replace <node-to-drain> with the name of your node you are draining
kubectl drain <node-to-drain> --ignore-daemonsets
```
### Upgrade kubelet and kubectl 
+ Upgrade the kubelet and kubectl on all worker nodes:
```
# replace x in 1.19.x-0 with the latest patch version
yum install -y kubelet-1.19.x-0 kubectl-1.19.x-0 --disableexcludes=kubernetes
```
+ Restart the kubelet
```
sudo systemctl daemon-reload && sudo systemctl restart kubelet
```

### Uncordon the node
+ Bring the node back online by marking it schedulable:
```
# replace <node-to-drain> with the name of your node
kubectl uncordon <node-to-drain>
```

## Verify the status of the cluster
After the kubelet is upgraded on all nodes verify that all nodes are available again
by running the following command from anywhere kubectl can access the cluster:
```
kubectl get nodes
```
And finally the upgrade process is done!!


## Referance links
+ https://v1-19.docs.kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/
+ https://elastisys.com/backup-kubernetes-how-and-why/
