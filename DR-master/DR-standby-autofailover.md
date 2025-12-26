# **Standby Master Automation – Disaster Recovery Document**

*(Cold Standby Control Plane with ETCD Snapshot Restore + Automatic Failover)*

---

## **Cluster Topology**

| Node Name | Role                  | IP Address     |
| --------- | --------------------- | -------------- |
| master-1  | Primary Control Plane | 192.168.30.200 |
| worker-1  | Worker Node           | 192.168.30.201 |
| master-2  | Standby Control Plane | 192.168.30.202 |

---

## **Overview**

* Take **etcd snapshot** from `master-1`
* Restore snapshot on `master-2`
* Initialize Kubernetes control-plane on `master-2`
* Rejoin worker node
* Validate:

  * Existing pods still run
  * New pods can be created
  * CNI (Calico) works **without re-applying manifests**

---

## **Objective**

To implement a **fully automated cold-standby disaster recovery solution** for Kubernetes where:

* `master-1` runs as the **active control-plane**
* `master-2` continuously **monitors master-1**
* If **both API server and machine fail for > 5 minutes**
* `master-2` **automatically restores ETCD**, promotes itself as **new control-plane**, and **rejoins worker nodes**

---

## **Architecture Overview**

```
master-1 (Active)
   |
   |  ETCD Snapshot → MinIO
   |
master-2 (Standby + Monitor)
   |
   |  API + Host Health Checks
   |
   └──> Auto Trigger DR Restore Script
```

---

## **Phase 1: Primary Master (master-1) – ETCD Backup**

### **Node:** `master-1 (192.168.30.200)`

### **1. Install ETCD Client**

```bash
sudo apt install etcd-client
```

### **2. Verify Cluster Status**

```bash
kubectl get nodes
```

Expected Output:

```text
master-1   Ready    control-plane   v1.29.15
worker-1   Ready    <none>          v1.29.15
```

---

### **3. Create Test Workload**

```bash
kubectl run master1-pod --image=nginx
```

```bash
kubectl get pods
```

Expected Output:

```text
master1-pod   1/1   Running
```

---

### **4. Take ETCD Snapshot**

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

---

### **5. Upload Snapshot to MinIO**

* Snapshot: `myback.db`
* MinIO Endpoint: `192.168.30.107`
* Uploaded manually via MinIO UI

---

## **Phase 2: Standby Master Preparation (master-2)**

### **Node:** `master-2 (192.168.30.202)`

* Kubernetes prerequisites already installed
* Ready for `kubeadm init`

### **Install ETCD Client**

```bash
sudo apt install etcd-client
```

---

## **Phase 3: MinIO Client Configuration (master-2)**

### **Install MinIO Client**

```bash
wget https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc
sudo mv mc /usr/local/bin/
```

### **Configure MinIO Alias**

```bash
mc alias set myminio http://192.168.30.107:9000 minioadmin minioadmin
mc ls myminio
```

---

## **Phase 4: SSH Configuration (master-2 → worker-1)**

```bash
sudo apt install openssh-server openssh-client -y
ssh-keygen -t rsa
ssh-copy-id kube@192.168.30.201
```

---

## **Phase 5: Kubernetes Init Configuration**

### **File:** `/home/kube/master-2-config.yaml`

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

## **Phase 6: Worker Node Sudo Configuration**

### **Node:** `worker-1 (192.168.30.201)`

```bash
sudo visudo
```

Add:

```text
kube ALL=(ALL) NOPASSWD:ALL
Defaults:kube !requiretty
```

---

## **Phase 7: Disaster Recovery Restore Script (master-2)**

📌 *This script restores ETCD, initializes control-plane, and rejoins workers.*

**File:**
`/home/kube/restore/dr_restore.sh`

### Script Content

```bash
#!/bin/bash
set -e

### =========================
### VARIABLES
### =========================
SNAPSHOT_NAME="myback.db"
MINIO_ALIAS="myminio"
MINIO_BUCKET="dr-bucket"
MINIO_PATH="${MINIO_ALIAS}/${MINIO_BUCKET}/${SNAPSHOT_NAME}"

ETCD_DATA_DIR="/var/lib/etcd"
LOCAL_SNAPSHOT_DIR="/home/kube/ETCD"
CONFIG_FILE="/home/kube/master-2-config.yaml"

MASTER_IP="192.168.30.202"
WORKERS=("192.168.30.201")

K8S_VERSION="v1.29.15"

### =========================
### FUNCTIONS
### =========================
log() {
  echo -e "\n\e[1;32m[INFO]\e[0m $1"
}

### =========================
### STEP 1: Download Snapshot
### =========================
log "Creating snapshot directory"
mkdir -p ${LOCAL_SNAPSHOT_DIR}

log "Downloading ETCD snapshot from MinIO"
mc cp ${MINIO_PATH} ${LOCAL_SNAPSHOT_DIR}/${SNAPSHOT_NAME}

### =========================
### STEP 2: Restore ETCD
### =========================
log "Stopping kubelet"
#sudo systemctl stop kubelet

log "Cleaning old ETCD data"
sudo rm -rf ${ETCD_DATA_DIR}

log "Restoring ETCD snapshot"
sudo ETCDCTL_API=3 etcdctl snapshot restore \
  ${LOCAL_SNAPSHOT_DIR}/${SNAPSHOT_NAME} \
  --name master \
  --initial-cluster master=https://${MASTER_IP}:2380 \
  --initial-cluster-token etcd-dr-cluster \
  --initial-advertise-peer-urls https://${MASTER_IP}:2380 \
  --data-dir ${ETCD_DATA_DIR}

### =========================
### STEP 3: Init Control Plane
### =========================
log "Initializing Kubernetes control-plane"
sudo kubeadm init \
  --config ${CONFIG_FILE} \
  --ignore-preflight-errors=DirAvailable--var-lib-etcd

        ### =========================
### STEP 4: Configure kubectl
### =========================
log "Configuring kubectl"
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

### =========================
### STEP 5: Get Join Command
### =========================
log "Generating join command"
JOIN_CMD=$(kubeadm token create --print-join-command)

### =========================
### STEP 6: Reset & Rejoin Workers
### =========================
for WORKER in "${WORKERS[@]}"; do
  log "Resetting worker node: $WORKER"

  ssh kube@$WORKER <<EOF
sudo kubeadm reset -f
sudo systemctl stop kubelet
sudo rm -rf /etc/kubernetes/*
sudo rm -rf /var/lib/kubelet/*
sudo rm -rf /var/lib/cni/*
sudo rm -rf /etc/cni/*
sudo systemctl restart containerd
sudo systemctl start kubelet
EOF

log "Waiting 3-4 minutes before joining worker $WORKER"
sleep 240

  log "Rejoining worker node: $WORKER"
  ssh kube@$WORKER "sudo ${JOIN_CMD}"
done

### =========================
### FINAL STATUS
### =========================
log "Disaster Recovery Completed Successfully!"
kubectl get nodes
kubectl get pods -A
```





---

# **Phase 8: FULL AUTOMATION – Standby Monitoring & Auto Failover**

## **Standby with Monitor Script**

### **Node:** `master-2 (192.168.30.202)`

This component ensures **automatic disaster recovery** without manual intervention.

---

## **8.1 Create Monitoring Script**

```bash
nano /home/kube/monitor_master.sh
```

### **Script Content**

```bash
#!/bin/bash

# =========================
# Master-1 Monitor Script
# Systemd-friendly version
# =========================

# Add standard paths for systemd
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

# ------------------------------
# CONFIGURATION
# ------------------------------
MASTER_IP="192.168.30.200"
DR_SCRIPT="/home/kube/restore/dr_restore.sh"
CHECK_INTERVAL=20     # seconds between checks
FAIL_LIMIT=10         # consecutive failures to trigger DR
FAIL_WINDOW=300       # 5 minutes window in seconds
LOG_FILE="/var/log/master-monitor.log"

# Absolute paths for commands
CURL=/usr/bin/curl
PING=/bin/ping
SUDO=/usr/bin/sudo
DATE=/bin/date
TEE=/usr/bin/tee

# ------------------------------
# VARIABLES
# ------------------------------
api_fail_count=0
host_fail_count=0
first_fail_time=0

# Ensure log file exists
touch $LOG_FILE
$DATE +"%F %T [INFO] Master monitor started" | $TEE -a $LOG_FILE

# ------------------------------
# MONITOR LOOP
# ------------------------------
while true; do

  # --- Check API Server ---
  $CURL -k --max-time 3 https://$MASTER_IP:6443/healthz >/dev/null 2>&1
  if [ $? -ne 0 ]; then
    api_fail_count=$((api_fail_count + 1))
    $DATE +"%F %T [WARN] API FAIL ($api_fail_count/$FAIL_LIMIT)" | $TEE -a $LOG_FILE
  else
    api_fail_count=0
  fi

  # --- Check Machine Reachability ---
  $PING -c 1 -W 2 $MASTER_IP >/dev/null 2>&1
  if [ $? -ne 0 ]; then
    host_fail_count=$((host_fail_count + 1))
    $DATE +"%F %T [WARN] MACHINE FAIL ($host_fail_count/$FAIL_LIMIT)" | $TEE -a $LOG_FILE
  else
    host_fail_count=0
  fi

  # --- Start timer on first failure ---
  if [ $api_fail_count -eq 1 ] || [ $host_fail_count -eq 1 ]; then
    first_fail_time=$(date +%s)
  fi

  # --- Check if DR should be triggered ---
  now=$(date +%s)
  elapsed=$(( now - first_fail_time ))

  if [ $api_fail_count -ge $FAIL_LIMIT ] && [ $host_fail_count -ge $FAIL_LIMIT ] && [ $elapsed -le $FAIL_WINDOW ]; then
    $DATE +"%F %T [ERROR] MASTER DEAD! Initiating DR..." | $TEE -a $LOG_FILE
    if [ -x "$DR_SCRIPT" ]; then
      $SUDO $DR_SCRIPT | $TEE -a $LOG_FILE
      $DATE +"%F %T [INFO] DR script completed. Continuing monitoring..." | $TEE -a $LOG_FILE
    else
      $DATE +"%F %T [ERROR] DR script not found or not executable!" | $TEE -a $LOG_FILE
    fi
    # Reset fail counters and timer after DR
    api_fail_count=0
    host_fail_count=0
    first_fail_time=0
  fi

  # --- Reset timer if elapsed > window ---
  if [ $elapsed -gt $FAIL_WINDOW ]; then
    api_fail_count=0
    host_fail_count=0
    first_fail_time=0
    $DATE +"%F %T [INFO] Failure window expired, counters reset" | $TEE -a $LOG_FILE
  fi

  sleep $CHECK_INTERVAL
done

```

---

### **Make Script Executable**

```bash
chmod +x /home/kube/monitor_master.sh
```

---

## **8.2 Systemd Service Configuration**

```bash
sudo nano /etc/systemd/system/master-monitor.service
```

```ini
[Unit]
Description=Monitor Primary Master and Trigger DR
After=network.target

[Service]
ExecStart=/bin/bash /home/kube/monitor_master.sh
Restart=always
User=kube

[Install]
WantedBy=multi-user.target
```

---

### **Enable & Start Monitoring**

```bash
sudo systemctl daemon-reexec
sudo systemctl daemon-reload
sudo systemctl enable master-monitor
sudo systemctl start master-monitor
```

---

### **Verify Monitoring**

```bash
sudo systemctl status master-monitor
tail -f /var/log/master-monitor.log
```

---

## **Failure Detection Logic (Summary)**

| Condition                | Action         |
| ------------------------ | -------------- |
| API FAIL < 5 mins        | Wait           |
| Host FAIL < 5 mins       | Wait           |
| API + Host FAIL > 5 mins | 🚀 Trigger DR  |
| Transient failure        | Reset counters |

---

### Final Outcome Snapshots

![Alt text](master-1.png)

![Alt text](master-2.png)

---

## Why Calico Worked Without Re-Applying Manifest

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

## **Final Result**

✅ Fully automated cold-standby Kubernetes DR
✅ ETCD snapshot restore from MinIO
✅ Standby master auto-promoted
✅ Worker node auto-rejoined
✅ Zero manual intervention

---

