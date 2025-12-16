# **Falco in Kubernetes**


## **What is Falco?**

Falco is an **open-source runtime security tool** created by **Sysdig**.
It provides **real-time threat detection** for Kubernetes by monitoring the behavior of:

* Containers
* Pods
* Nodes

Falco detects suspicious activity **while applications are running**, not just at deploy time.

---

## **How Falco Works (Syscall Monitoring)**

In Linux, when a program wants to perform a low-level operation such as:

* Reading or writing files
* Creating processes
* Opening network connections
* Accessing secrets
* Changing file permissions

It must request permission from the **Linux Kernel** using a **system call (syscall)**.

👉 **Falco monitors every syscall** made by every container running in the cluster.

---

## **Examples of Suspicious Activity Detected by Falco**

Falco immediately detects and alerts if a container:

* Executes `rm -rf /`
* Reads sensitive files like `/etc/passwd`
* Connects to unknown or suspicious IP addresses
* Runs unexpected or unauthorized commands
* Modifies system files
* Opens an interactive shell inside a production pod

---

## **Falco Architecture**

### **1. Kernel Module / eBPF**

* Runs inside the Linux kernel
* Captures system calls from containers and nodes

### **2. Falco Rules**

* Define what behavior is considered **malicious or suspicious**
* Example: *Containers should not write to `/etc`*

### **3. Falco Engine**

* Evaluates syscalls against the defined rules

### **4. Alerts**

* Generated when a rule is violated
* Alerts are sent via logs (and can be integrated with Slack, SIEM, etc.)

---

## **Deploying Falco in Kubernetes**

### **Step 1: Add Falco Helm Repository**

```bash
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update
```

---

### **Step 2: Install Falco Using Helm**

```bash
helm install --replace falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set tty=true
```

---

### **Step 3: Verify Falco Pods**

```bash
kubectl get pods -n falco
```

Expected output (example):

```text
falco-xxxxx   1/1   Running   0   30s
```

---

## **Create a Sample Pod for Testing**

Create a simple **nginx pod**:

```bash
kubectl run nginx-pod --image=nginx
```

Verify pod status:

```bash
kubectl get pods
```

Expected output:

```text
nginx-pod   1/1   Running   0   6s
```

---

## **Creating a Custom Falco Rule**

Create a file named **`falco-customrules.yaml`**:

```yaml
- rule: Read Passwd File
  desc: Detect reading of /etc/passwd inside a container
  condition: open_read and fd.name=/etc/passwd
  output: >
    SENSITIVE FILE READ: /etc/passwd
    (user=%user.name container=%container.id)
  priority: WARNING
  tags: [filesystem, security]
```

---

## **Apply the Custom Rule Using Helm**

```bash
helm upgrade --install falco falcosecurity/falco \
  -n falco \
  --create-namespace \
  --set falco.rulesFile=./falco-customrules.yaml
```

---

## **Trigger the Falco Rule**

Enter the nginx pod:

```bash
kubectl exec -it nginx-pod -- sh
```

Run the command that violates the rule:

```bash
cat /etc/passwd
```

---

## **View Falco Alerts**

Check Falco logs:

```bash
kubectl logs -n falco -l app.kubernetes.io/name=falco -f
```

---

## **Sample Falco Alert Output**

```text
12:31:49.459137273: Warning
SENSITIVE FILE READ: /etc/passwd
(user=root container=host)
container_id=host
container_name=host
```

---

## **Conclusion**

* Falco provides **runtime security** for Kubernetes
* It monitors **syscalls in real time**
* Uses **rules** to detect abnormal behavior
* Generates **immediate alerts** for security violations
* Helps detect attacks like container escapes, privilege abuse, and data exfiltration

---

