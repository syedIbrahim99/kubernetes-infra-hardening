# Kubernetes Control Plane Hardening

### **Network Access Restriction + kubectl Setup on Windows**

---

## **Environment**

* **Admin Windows Laptop IP:** `192.168.30.25`
* **Control Plane IP:** `192.168.30.83`

---

# **1. Control Plane Hardening Using UFW (Uncomplicated Firewall)**

### **Commands (Run on Control Plane Node):**

```bash
sudo ufw enable

sudo ufw allow from 192.168.30.25 to any port 6443
sudo ufw deny 6443

sudo ufw allow from 192.168.30.25 to any port 22
sudo ufw deny 22

sudo ufw status verbose
sudo ufw status numbered
sudo ufw delete 3        # use if needed to delete any rules
```

### **Result:**

Now **only the Admin laptop (`192.168.30.25`)** can:

* SSH into the control plane
* Make Kubernetes API calls on port 6443

All **other laptops in the same network are blocked**.

---

## **Example Tests**

### **From a Random Laptop (same network):**

```cmd
curl -k https://192.168.30.83:6443
```

**Output:**

```
curl: (28) Failed to connect to 192.168.30.83 port 6443 after 21008 ms
```

### **From the Admin Laptop:**

```cmd
curl -k https://192.168.30.83:6443
```

**Output:**

```json
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {},
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {},
  "code": 403
}
```

✔️ API server reachable
✔️ Unauthorized because no token used — this is normal
✔️ Random laptops **cannot connect** at all

Same applies for SSH:

* **Admin laptop → SSH allowed**
* **Random laptops → SSH blocked**

---

# **2. kubectl Installation on Windows**

### **Official Documentation:**

👉 [Install kubectl on Windows (Kubernetes Official Documentation)](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-kubectl-binary-on-windows-via-direct-download-or-curl)

---

### **Download kubectl (official method):** (CMD - C:\Users\Syedibrahim\Downloads)

```cmd
curl.exe -LO "https://dl.k8s.io/release/v1.34.0/bin/windows/amd64/kubectl.exe"
```

---

## **Move kubectl to System32 (run CMD as Administrator):**

```cmd
move C:\Users\Syedibrahim\Downloads\kubectl.exe C:\Windows\System32\
```

**Output:**

```
1 file(s) moved.
```

---

## **Verify Installation**

```cmd
kubectl version --client
```

**Output:**

```
Client Version: v1.34.0
Kustomize Version: v5.7.1
```

---

# **3. Configure Kubernetes Access on Windows**

### **Steps:**

1. Create a `.kube` directory:
   `C:\Users\Syedibrahim\.kube`
2. Copy the **control-plane kubeconfig** into this folder as:
   `C:\Users\Syedibrahim\.kube\config`

---

## **Test Cluster Access**

Navigate to:

```
C:\Users\Syedibrahim\Downloads
```

Run:

```cmd
kubectl get nodes
```

**Output (Success):**

```
NAME      STATUS   ROLES           AGE     VERSION
hardk8s   Ready    control-plane   4h37m   v1.32.10
worker    Ready    <none>          4h37m   v1.32.10
```

---

## **If Firewall Blocks the Windows Machine**

If UFW denies this IP:

```cmd
kubectl get nodes
```

**Output (Failure):**

```
Unable to connect to the server: dial tcp 192.168.30.83:6443:
connectex: A connection attempt failed because the connected party did not properly respond...
```

✔️ Meaning: The machine is not allowed in UFW
✔️ API communication blocked

---

# **conclusion**

* UFW rules ensure **only Admin laptop** can access

  * Kubernetes API (6443)
  * SSH (22)
* kubectl on Windows works only when the IP is **allowed** in UFW.
* Random laptops in same LAN **cannot** access the cluster API or SSH when restricted.

---

