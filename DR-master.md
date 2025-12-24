
# Kubernetes Manual Disaster Recovery (Cold Standby Master)

**ETCD Snapshot Restore + Control Plane Promotion**

---

## 1. Cluster Topology

| Node Name | Role                         | IP Address     |
| --------- | ---------------------------- | -------------- |
| master-1  | Control Plane (Primary)      | 192.168.30.200 |
| worker-1  | Worker Node                  | 192.168.30.201 |
| master-2  | Control Plane (Standby / DR) | 192.168.30.202 |

---

## 2. Objective

* Take **etcd snapshot** from `master-1`
* Restore snapshot on `master-2`
* Initialize Kubernetes control-plane on `master-2`
* Rejoin worker node
* Validate:

  * Existing pods still run
  * New pods can be created
  * CNI (Calico) works **without re-applying manifests**

---

## 3. Actions on master-1 (Primary Master)

### 3.1 Install etcd client

```bash
sudo apt install -y etcd-client
```

### 3.2 Verify cluster status

```bash
kubectl get nodes
```

Expected:

```
master-1   Ready    control-plane   5m8s    v1.29.15
worker-1   Ready    <none>          4m40s   v1.29.15
```

### 3.3 Create test workload

```bash
kubectl run master1-pod --image=nginx
kubectl get pods
```

Expected:

```
master1-pod   Running
```

---



## 4. Take ETCD Snapshot (master-1)

```bash
mkdir etcd-dir
cd etcd-dir
```

```bash
sudo ETCDCTL_API=3 etcdctl snapshot save myback.db \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

### 4.1 Copy snapshot to master-2

```bash
scp myback.db kube@192.168.30.202:/home/kube/ETCD
```

> **Note:** Create `/home/kube/ETCD` directory on `master-2` before copying.

---

## 5. Actions on master-2 (DR / Standby Master)

### 5.1 Install etcd client

```bash
sudo apt install -y etcd-client
```

---

### 5.2 Restore ETCD Snapshot

```bash
sudo ETCDCTL_API=3 etcdctl snapshot restore /home/kube/ETCD/myback.db \
  --name master \
  --initial-cluster master=https://192.168.30.202:2380 \
  --initial-cluster-token etcd-cluster-1 \
  --initial-advertise-peer-urls https://192.168.30.202:2380 \
  --data-dir /var/lib/etcd
```

Expected:

```
2025-12-24 10:57:52.704327 I | etcdserver/membership: added member cedde21026c0123c [https://192.168.30.202:2380] to cluster d9030b9d62e7bf88
```

---

## 6. kubeadm Configuration File (master-2)

📄 **File:** `/home/kube/master-2-config.yaml`

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

## 7. Initialize Control Plane on master-2

```bash
sudo kubeadm init --ignore-preflight-errors=DirAvailable--var-lib-etcd
```
* This above command doesn't need the yaml file which we created.

```bash
sudo kubeadm init \
  --config /home/kube/master-2-config.yaml \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd
```

Expected:

```bash
[init] Using Kubernetes version: v1.29.15
[preflight] Running pre-flight checks
        [WARNING Hostname]: hostname "master" could not be reached
        [WARNING Hostname]: hostname "master": lookup master on 127.0.0.53:53: server misbehaving
        [WARNING DirAvailable--var-lib-etcd]: /var/lib/etcd is not empty
[preflight] Pulling images required for setting up a Kubernetes cluster
[preflight] This might take a minute or two, depending on the speed of your internet connection
[preflight] You can also perform this action in beforehand using 'kubeadm config images pull'
W1224 11:00:46.065567   10819 checks.go:835] detected that the sandbox image "registry.k8s.io/pause:3.8" of the container runtime is inconsistent with that used by kubeadm. It is recommended that using "registry.k8s.io/pause:3.9" as the CRI sandbox image.
[certs] Using certificateDir folder "/etc/kubernetes/pki"
[certs] Generating "ca" certificate and key
[certs] Generating "apiserver" certificate and key
[certs] apiserver serving cert is signed for DNS names [kubernetes kubernetes.default kubernetes.default.svc kubernetes.default.svc.cluster.local master] and IPs [10.96.0.1 192.168.30.202]
[certs] Generating "apiserver-kubelet-client" certificate and key
[certs] Generating "front-proxy-ca" certificate and key
[certs] Generating "front-proxy-client" certificate and key
[certs] Generating "etcd/ca" certificate and key
[certs] Generating "etcd/server" certificate and key
[certs] etcd/server serving cert is signed for DNS names [localhost master] and IPs [192.168.30.202 127.0.0.1 ::1]
[certs] Generating "etcd/peer" certificate and key
[certs] etcd/peer serving cert is signed for DNS names [localhost master] and IPs [192.168.30.202 127.0.0.1 ::1]
[certs] Generating "etcd/healthcheck-client" certificate and key
[certs] Generating "apiserver-etcd-client" certificate and key
[certs] Generating "sa" key and public key
[kubeconfig] Using kubeconfig folder "/etc/kubernetes"
[kubeconfig] Writing "admin.conf" kubeconfig file
[kubeconfig] Writing "super-admin.conf" kubeconfig file
[kubeconfig] Writing "kubelet.conf" kubeconfig file
[kubeconfig] Writing "controller-manager.conf" kubeconfig file
[kubeconfig] Writing "scheduler.conf" kubeconfig file
[etcd] Creating static Pod manifest for local etcd in "/etc/kubernetes/manifests"
[control-plane] Using manifest folder "/etc/kubernetes/manifests"
[control-plane] Creating static Pod manifest for "kube-apiserver"
[control-plane] Creating static Pod manifest for "kube-controller-manager"
[control-plane] Creating static Pod manifest for "kube-scheduler"
[kubelet-start] Writing kubelet environment file with flags to file "/var/lib/kubelet/kubeadm-flags.env"
[kubelet-start] Writing kubelet configuration to file "/var/lib/kubelet/config.yaml"
[kubelet-start] Starting the kubelet
[wait-control-plane] Waiting for the kubelet to boot up the control plane as static Pods from directory "/etc/kubernetes/manifests". This can take up to 4m0s
[apiclient] All control plane components are healthy after 20.517732 seconds
[upload-config] Storing the configuration used in ConfigMap "kubeadm-config" in the "kube-system" Namespace
[kubelet] Creating a ConfigMap "kubelet-config" in namespace kube-system with the configuration for the kubelets in the cluster
[upload-certs] Skipping phase. Please see --upload-certs
[mark-control-plane] Marking the node master as control-plane by adding the labels: [node-role.kubernetes.io/control-plane node.kubernetes.io/exclude-from-external-load-balancers]
[mark-control-plane] Marking the node master as control-plane by adding the taints [node-role.kubernetes.io/control-plane:NoSchedule]
[bootstrap-token] Using token: rj7t7q.tvtinsnt3yt3ikbt
[bootstrap-token] Configuring bootstrap tokens, cluster-info ConfigMap, RBAC Roles
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to get nodes
[bootstrap-token] Configured RBAC rules to allow Node Bootstrap tokens to post CSRs in order for nodes to get long term certificate credentials
[bootstrap-token] Configured RBAC rules to allow the csrapprover controller automatically approve CSRs from a Node Bootstrap Token
[bootstrap-token] Configured RBAC rules to allow certificate rotation for all node client certificates in the cluster
[bootstrap-token] Creating the "cluster-info" ConfigMap in the "kube-public" namespace
[kubelet-finalize] Updating "/etc/kubernetes/kubelet.conf" to point to a rotatable kubelet client certificate and key
[addons] Applied essential addon: CoreDNS
[addons] Applied essential addon: kube-proxy

Your Kubernetes control-plane has initialized successfully!

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config

Alternatively, if you are the root user, you can run:

  export KUBECONFIG=/etc/kubernetes/admin.conf

You should now deploy a pod network to the cluster.
Run "kubectl apply -f [podnetwork].yaml" with one of the options listed at:
  https://kubernetes.io/docs/concepts/cluster-administration/addons/

Then you can join any number of worker nodes by running the following on each as root:

kubeadm join 192.168.30.202:6443 --token rj7t7q.tvtinsnt3yt3ikbt \
        --discovery-token-ca-cert-hash sha256:b8b770f93ef7191372b8c57eb9a30d089daf8bddd614228f78738f993781bd50
```

* With both commands will be able to see the pods, service's nginx page in browser.

---

### 7.1 Configure kubectl

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

---

## 8. Cluster State After Restore

```bash
kubectl get nodes
```

Expected:

```
master     Ready      control-plane   104s   v1.29.15
master-1   NotReady   control-plane   14m    v1.29.15
worker-1   NotReady   <none>          14m    v1.29.15
```

```bash
kubectl get pods
```

Expected:

```
master1-pod   Running
```

✅ **Workload survived ETCD restore**

---

## 9. Rejoin Worker Node (worker-1)

### 9.1 Cleanup old state

```bash
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo rm -rf /etc/kubernetes/*
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /var/lib/cni/*
sudo rm -rf /etc/cni/*
sudo systemctl restart containerd
sudo systemctl start kubelet
```

### 9.2 Join cluster

```bash
sudo kubeadm join 192.168.30.202:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

---

## 10. Final Cluster Validation

```bash
kubectl get nodes
```

Expected:

```
master     Ready      control-plane   104s   v1.29.15
master-1   NotReady   control-plane   14m    v1.29.15
worker-1   NotReady   <none>          14m    v1.29.15
```

```bash
kubectl get pods
```

Expected:

```
master1-pod   Running
```

---

## 11. Create New Pod (Post-DR Validation)

```bash
kubectl run master2-pod --image=nginx
kubectl get pods
```

Expected:

```
master1-pod   Running
master2-pod   Running
```

---

### Final Outcome Snaphots

![Alt text](master-1.png)

![Alt text](master-2.png)



---

## 12. Why Calico Worked Without Re-Applying Manifest

### Timeline Explanation

| Time | Event                                      |
| ---- | ------------------------------------------ |
| T0   | Fresh `master-2`, no CNI binaries          |
| T1   | ETCD snapshot restored                     |
| T2   | kubeadm init                               |
| T3   | `calico-node` DaemonSet scheduled          |
| T4   | Calico installs CNI binaries automatically |

---

### How Calico Does This

* `calico-node` is a **DaemonSet**
* Runs in **privileged mode**
* Uses **hostPath mounts**:

  * `/opt/cni/bin`
  * `/etc/cni/net.d`
* Automatically installs:

  * CNI binaries
  * CNI config
* No manual CNI apply required

---

