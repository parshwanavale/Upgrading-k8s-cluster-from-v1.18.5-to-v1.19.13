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
