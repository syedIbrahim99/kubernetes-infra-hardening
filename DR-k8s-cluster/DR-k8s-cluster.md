

# Kubernetes whole Cluster Disaster Recovery using ETCD Snapshot Restore

## Overview

This document explains how to **restore a Kubernetes cluster using an ETCD snapshot** when the original cluster (both master and workers) has completely failed.

The scenario demonstrates restoring the cluster state from **Cluster-A** into **Cluster-B** using an ETCD snapshot backup.

Key observations during testing:

* All Kubernetes resources stored in ETCD were restored.
* Pods running on the old worker node were automatically **rescheduled to the new worker node**.
* **PersistentVolume and PersistentVolumeClaim objects were restored**, but **actual PV data was not restored**.

---

# 1. Original Cluster (Cluster-A)

## Cluster-A Machines

| Node Name | Role          | IP Address     |
| --------- | ------------- | -------------- |
| master-1  | Control Plane | 192.168.30.150 |
| worker-1  | Worker Node   | 192.168.30.151 |

---

## Steps Performed in Cluster-A

### 1. Initialize Kubernetes Cluster

The cluster was initialized using `kubeadm`.

### 2. Deploy Applications

Two deployments were created:

* **nginx deployment**
* **nginx deployment with Persistent Volume**

---

### 3. Install ETCD Client

```bash
sudo apt install -y etcd-client
```

---

### 4. Take ETCD Snapshot

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save cluster-backup.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

This creates the snapshot file:

```
cluster-backup.db
```

---

### 5. Store Snapshot Backup

The snapshot file is copied outside the cluster and stored safely.

This backup will be used to restore the cluster.

---

# 2. Disaster Scenario

Assume the following situation:

* **Cluster-A completely crashed**
* Only the **ETCD snapshot file** is available

```
cluster-backup.db
```

Goal:

Restore the cluster into a **new environment (Cluster-B)**.

---

# 3. New Cluster (Cluster-B)

## Cluster-B Machines

| Node Name | Role          | IP Address     |
| --------- | ------------- | -------------- |
| master-2  | Control Plane | 192.168.30.160 |
| worker-2  | Worker Node   | 192.168.30.161 |

---

# 4. Restore ETCD Snapshot in Cluster-B

## Step 1 — Install Kubernetes prerequisites

Install all Kubernetes prerequisites on:

* master-2
* worker-2

---

## Step 2 — Install ETCD Client

```bash
sudo apt install -y etcd-client
```

---

## Step 3 — Copy Snapshot to Master

Copy the backup file to **master-2**.

Example:

```bash
scp cluster-backup.db kube@192.168.30.160:/home/kube/etcd-dir/
```

---

## Step 4 — Restore ETCD Snapshot

Run the restore command on **master-2**:

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /home/kube/etcd-dir/cluster-backup.db \
  --name master \
  --initial-cluster master=https://192.168.30.160:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://192.168.30.160:2380 \
  --data-dir /var/lib/etcd
```

This restores the ETCD data into:

```
/var/lib/etcd
```

---

# 5. Initialize Control Plane using Restored ETCD

Create configuration file:

```
/home/kube/master-2-config.yaml
```

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: InitConfiguration
nodeRegistration:
  name: master
  criSocket: unix:///run/containerd/containerd.sock

---
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: v1.29.15
etcd:
  local:
    dataDir: /var/lib/etcd
networking:
  podSubnet: 10.244.0.0/16
```

---

## Run kubeadm init

Two options are available.

### Option 1 (Using Config File)

```bash
sudo kubeadm init \
  --config /home/kube/master-2-config.yaml \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

---

### Option 2 (Without Config File)

```bash
sudo kubeadm init \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

- This document is tested with option 2.

Explanation:

* Since the **ETCD snapshot already contains cluster configuration**, the pod subnet will automatically match the restored configuration.
* The command uses the **hostname as node name**.

Important:

Ensure the **new master hostname is different from the old master**.

Example:

```
Old master → master-1
New master → master-2
```

---

# 6. Node Status After Restore

Initial state:

```
kubectl get nodes
```

```
NAME       STATUS     ROLES           VERSION
master-1   Ready      control-plane   v1.29.15
master-2   NotReady   control-plane   v1.29.15
worker-1   Ready      <none>          v1.29.15
```

After some time:

```
NAME       STATUS
master-1   NotReady
master-2   Ready
worker-1   NotReady
```

This occurs because:

* Old nodes are still present in the restored ETCD state.
* But they are unreachable.

---

# 7. Join New Worker Node

Login to **worker-2** and run the **kubeadm join command**.

Example:

```bash
kubeadm join <master-ip>:6443 --token <token> ...
```

---

# 8. Node Status After Worker Join

```
kubectl get nodes
```

```
NAME       STATUS
master-1   NotReady
master-2   Ready
worker-1   NotReady
worker-2   NotReady
```

After **Calico and CoreDNS pods start running**, the node becomes ready.

```
worker-2   Ready
```

---

# 9. Pod Rescheduling Behavior

Pods originally running on **worker-1** appear in **Terminating** state.

Example:

```
nginx-deploy-1-d565df4fc-7qj6q   Terminating   worker-1
```

New pods get scheduled on **worker-2**:

```
nginx-deploy-1-d565df4fc-bzbjl   Running   worker-2
```

Reason:

* `worker-1` is **NotReady**
* Kubernetes scheduler automatically **recreates pods on available nodes**

---

# 10. Accessing Applications

After pod rescheduling completes:

* Nginx applications become accessible again
* Services continue functioning normally

---

# 11. Handling Old Nodes

Old nodes from the snapshot may still appear.

Example:

```
master-1
worker-1
```

These can be managed using:

### Cordon Node

```bash
kubectl cordon <node-name>
```

### Delete Node

```bash
kubectl delete node <node-name>
```

---

# 12. Persistent Volume Behavior

After restoration:

| Resource              | Restored |
| --------------------- | -------- |
| PersistentVolume      | Yes      |
| PersistentVolumeClaim | Yes      |
| PV Data               | No       |

Reason:

ETCD only stores **cluster metadata**, not the actual storage data.

Actual volume data depends on:

* NFS
* EBS
* Ceph
* Local disk

---

# 13. Pod Subnet Explanation

During cluster creation, a **pod subnet** is defined.

Example from automation playbook: (which Ashwin Gave)

```yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
networking:
  podSubnet: "10.244.0.0/16"
```

---

## How Calico Uses the Pod Subnet

When Calico is installed:

```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

Calico checks:

1. If cluster already has a pod subnet → use it
2. Otherwise → use default subnet

Default Calico subnet:

```
192.168.0.0/16
```

---

# 14. Why We Use 10.244.0.0/16

Example network environments:

```
Office Network → 192.168.30.0/24
Home Network   → 192.168.1.0/24
```

If Calico used:

```
192.168.0.0/16
```

Possible conflict:

Pods could receive IPs overlapping with the office/home network.

Example:

```
Pod IP → 192.168.1.25
Home Router → 192.168.1.1
```

This causes **routing conflicts**.

Using:

```
10.244.0.0/16
```

avoids these conflicts because it is a **dedicated private subnet for pod networking**.

---

# 15. Key Observations

* ETCD snapshot successfully restores cluster state
* Old nodes remain in ETCD until removed
* Pods are automatically rescheduled when nodes become unavailable
* PV objects are restored but storage data is not
* Pod subnet configuration persists through ETCD restore

---


