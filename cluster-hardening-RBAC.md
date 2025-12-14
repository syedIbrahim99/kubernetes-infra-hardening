# k8s-infra-hardening Document

---

## Users
Two users are involved in this setup:

- **kube** – Admin user  
- **developer** – User with limited permissions

---

## Overview
This document explains how to provide **restricted Kubernetes access** to a developer user using:
- ServiceAccount
- RBAC (RoleBinding with `view` access)
- Separate kubeconfig
- OS-level hardening (removing `sudo` and `su` access)

> 

---

## Steps


## kube User

### Create Namespace
```bash
kubectl create namespace dev
```
---

## Service Account

### `sa-dev.yaml`

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-dev
  namespace: autox-dev
```

---

## RBAC Configuration

### `rb-dev.yaml`

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: view-binding-dev
  namespace: autox-dev
subjects:
- kind: ServiceAccount
  name: sa-dev
  namespace: autox-dev
roleRef:
  kind: ClusterRole
  name: view
  apiGroup: rbac.authorization.k8s.io
```

---

## Service Account Token

### `dev-sa-secret.yaml`

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sa-token-dev
  namespace: autox-dev
  annotations:
    kubernetes.io/service-account.name: sa-dev
type: kubernetes.io/service-account-token
```

---

## Verify Secret

```bash
kubectl -n autox-dev get secret
```

---

## Token Extraction

```bash
kubectl describe secret -n autox-dev sa-token-dev
```

➡️ This will show the token value.

---

## CA Certificate

```bash
kubectl -n autox-dev get secret sa-token-dev -o jsonpath='{.data.ca\.crt}' | base64 -d > ca.crt
```

➡️ This creates a file named `ca.crt`.

---

## API Server URL

```bash
kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}'
```

---

## Configure Cluster in kubeconfig

```bash
kubectl config set-cluster my-cluster \
--server=https://192.168.1.150:6443 \
--certificate-authority=ca.crt \
--embed-certs=true \
--kubeconfig=dev.kubeconfig
```

---

## Set Credentials

> Paste the token obtained from the `sa-token-dev` secret.

```bash
kubectl config set-credentials dev \
--token=eyJhbGciOiJSUzI1NiIsImtpZCI6InRxWFFxdUJqVTRiaGRETnN1akVhQXc3UDlwczd6dzhYOHVrUkN0UlZsWDgifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJkZXYiLCJrdWJlcm5ldGVzLmlvL3NlcnZpY2VhY2NvdW50L3NlY3JldC5uYW1lIjoiZGV2LXNhLXRva2VuIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQubmFtZSI6ImRldi1zYSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50LnVpZCI6IjM2YzlhOTc2LTViMTQtNGNmNC1iZDc0LTZmNTM4ZTE4NDg0MSIsInN1YiI6InN5c3RlbTpzZXJ2aWNlYWNjb3VudDpkZXY6ZGV2LXNhIn0.Dx-rZ7LPTgQWCu14_s74vhs-7Cy1O69P4TzJkMDxIT-YWiA9GoDkBwa2cQzRHs2fcfV1iRY0z1n_ycg70cTJ_EKSISQF1MgJLSJU7DPxgZqspV_uPH7d5tTO-EwHP46WY2mBHuX0ZgPh-DQxxigBiIwwMeHyW-IlqCc2iZOwkjRwmN7E2j3mCxkZDz9GZS3v-X9NrN_ymmy6yNfWmIHq5VGHMnC-2f8BmAFM7DCrrDa9vQ1dXiyQWe5sl0HlMmG1GT3w3h__eDqkTdvjEJ9y994vGsT9mEf8eYUiHFyY8PpSXacIyMUca7gRZzCfMgOzrfQ2dBT8Z_7a6CCGDFRnKA \
--kubeconfig=dev.kubeconfig
```

---

## Set Context

```bash
kubectl config set-context dev-context \
--cluster=my-cluster \
--namespace=autox-dev \
--user=dev \
--kubeconfig=dev.kubeconfig
```

---

## File Ownership & Access

### kube User

```bash
sudo cp dev.kubeconfig /home/kube/
```

### developer User

```bash
sudo chown $USER:$USER dev.kubeconfig
```

---

## Use Context (Developer)

```bash
kubectl config use-context dev-context --kubeconfig=dev.kubeconfig
```

```bash
kubectl --kubeconfig=dev.kubeconfig get pods -n autox-dev
```

---

## OS-Level Hardening

* Remove **sudo** access for `developer`
* Remove **su** command permission for `developer`

---

## Remove `su` Permission

### kube User

```bash
sudo nano /etc/pam.d/su
```

➡️ Uncomment the following line (around line 15):

```text
# auth       required   pam_wheel.so
```

---

## Result

* Developer has **read-only access** to a specific namespace
* No cluster-admin or node access
* No `sudo` or `su` privileges
* Kubernetes control plane access is hardened

---

```
```
