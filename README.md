# GitOps Based Auto Deployment with ArgoCD
GitOps-based Kubernetes App Deployment with ArgoCD

> **Week 2 - DevOps Practice Exercise**
> This hands-on project is part of a DevOps learning series. In Week 2, we focus on implementing a GitOps workflow using ArgoCD to automate continuous deployments to Kubernetes clusters. The goal is to learn how to set up isolated **Staging** and **Production** environments, manage infrastructure declaratively, and enable safe, repeatable deployments via Git.

### 📸 ArgoCD Dashboard View

<img width="1270" alt="image" src="https://github.com/user-attachments/assets/763e0482-9813-42bb-b1a8-f04eb79d91f1" />


## 🔧 Prerequisites

Before you begin, make sure the following tools are installed and configured:

* `kubectl` client installed along with Docker runtime ([Docker Desktop](https://www.docker.com/products/docker-desktop/) or [Rancher Desktop](https://rancherdesktop.io/))
* Docker runtime (e.g., Docker Desktop or Rancher Desktop)
* A working Kubernetes cluster  
  <br>  
  👉 Use [these instructions](https://kind.sigs.k8s.io/docs/user/quick-start/) to set up a local cluster with **KIND**  
  <br>
  Example setup:
  ```bash
  git clone https://github.com/initcron/k8s-code.git
  cd k8s-code/helper/kind
  kind create cluster --config kind-three-node-cluster.yaml
  ```
  Example output:
  ```
  Creating cluster "kind" ...
   ✓ Ensuring node image (kindest/node:v1.33.1) 🖼
   ✓ Preparing nodes 📦 📦 📦
   ✓ Writing configuration 📜
   ✓ Starting control-plane 🕹️
   ✓ Installing CNI 🔌
   ✓ Installing StorageClass 💾
   ✓ Joining worker nodes 🚜
  Set kubectl context to "kind-kind"
  You can now use your cluster with:

  kubectl cluster-info --context kind-kind

  Thanks for using kind! 😊
  ```
### ✅ KIND Cluster Validation

After creating your KIND cluster, validate it with the following commands and expected output:

```bash
➜  kind git:(master) kind get clusters
kind
➜  kind git:(master) kubectl cluster-info --context kind-kind
Kubernetes control plane is running at https://127.0.0.1:53765
CoreDNS is running at https://127.0.0.1:53765/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

```bash

➜  kind git:(master) kubectl get nodes
NAME                 STATUS   ROLES           AGE     VERSION
kind-control-plane   Ready    control-plane   2m54s   v1.33.1
kind-worker          Ready    <none>          2m45s   v1.33.1
kind-worker2         Ready    <none>          2m45s   v1.33.1

```
* Git installed
* A terminal (e.g., iTerm, PowerShell, etc.)

## 🚀 Fork and Clone the Project

   ```bash
   git clone https://github.com/sfd-cicd/vote-deploy.git 
   cd vote-deploy
   ```


## 🚀 Install Argo CD

We'll install Argo CD in the `argocd` namespace and expose it via `NodePort` for local access.

### 1️⃣ Create Namespace

```bash
kubectl create namespace argocd
```

### 2️⃣ Install Argo CD Components

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 3️⃣ Wait for Pods to Be Ready

```bash
kubectl get pods -n argocd

➜  Weekly Nano Project 2 kubectl get pods -n argocd
NAME                                                READY   STATUS    RESTARTS      AGE
argocd-application-controller-0                     1/1     Running   0             97s
argocd-applicationset-controller-655cc58ff8-qdk72   1/1     Running   0             97s
argocd-dex-server-7d9dfb4fb8-dwptt                  1/1     Running   2 (46s ago)   97s
argocd-notifications-controller-6c6848bc4c-k57rv    1/1     Running   0             97s
argocd-redis-656c79549c-phdzn                       1/1     Running   0             97s
argocd-repo-server-856b768fd9-bm5jh                 1/1     Running   0             97s
argocd-server-99c485944-fz2cl                       1/1     Running   0             97s
```

Wait until all pods show `STATUS: Running`.

### 4️⃣ Expose Argo CD API Server (NodePort)

## ❌ Why NodePort Does Not Work with KIND

You may try this:

```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "NodePort"}}'
```

And attempt to access:

```
http://localhost:<NodePort>
```

But it **won’t work** because:

* KIND runs Kubernetes nodes inside Docker containers.
* `NodePort` exposes the service on the node (Docker container), **not your host**.
* So `localhost:<NodePort>` is **not reachable** without extra Docker port mappings.

## ✅ Port-Forward Argo CD API Server

To reliably access the Argo CD UI on KIND:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:80
```

🔁 Keep the terminal open while using the UI.

### Logs for reference
```bash

➜  > kubectl port-forward svc/argocd-server -n argocd 8080:80

Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
Handling connection for 8080
```

Then open:

```
http://localhost:8080
```

### 5️⃣ Get Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login with:

* **Username:** `admin`
* **Password:** (output from above)

### 🚀 Setup Deployment Repository

This section sets up a GitOps-style deployment using ArgoCD for a sample vote app, with **Staging** and **Production** environments.

---

### ✅ Prerequisites

* ArgoCD is installed and running in your cluster.
* ArgoCD UI is accessible via `kubectl port-forward svc/argocd-server -n argocd 8080:80`
* You have forked the deployment repo: [sfd-cicd/vote-deploy](https://github.com/sfd-cicd/vote-deploy)

---

### 1. Clone Your Fork

```bash
git clone https://github.com/<your-username>/vote-deploy.git
cd vote-deploy
```

---

### 2. Directory Structure

Your repo should look like:

```
vote-deploy/
├── base/
│   ├── deployment.yaml
│   ├── kustomization.yaml
│   └── service.yaml
├── prod/
│   ├── kustomization.yaml
│   └── service.yaml
└── staging/
    ├── kustomization.yaml
    └── service.yaml
```

---

### 3. Create Namespaces for Staging and Prod

```bash
kubectl create namespace staging
kubectl create namespace prod
```

```
➜  vote-deploy git:(main) ✗ kubectl get ns
NAME                 STATUS   AGE
argocd               Active   75m
default              Active   118m
kube-node-lease      Active   118m
kube-public          Active   118m
kube-system          Active   118m
local-path-storage   Active   118m
prod                 Active   47m
staging              Active   47m
```
---

### 4. Setup ArgoCD Applications via UI

#### ArgoCD UI

1. Go to `http://localhost:8080`
2. Login with default credentials:
   `admin` / (password from: `kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d`)
3. Click **New App**
4. Fill in:

   * App Name: `vote-staging`
   * Project: `default`
   * Sync Policy: Manual
   * Repo URL: your GitHub fork URL
   * Path: `staging`
   * Namespace: `staging`
<img width="499" alt="image" src="https://github.com/user-attachments/assets/0931b969-9432-4757-b543-c8d1f1a5732b" />
  
5. Repeat for `vote-prod`, using path `prod` and namespace `prod`.
<img width="503" alt="image" src="https://github.com/user-attachments/assets/5f5f7fc8-70a7-49a0-b698-d45981dcb3e5" />

### 5. Expose the App (NodePort)

Check service:

```bash

➜  ~ kubectl get all -n staging
NAME                        READY   STATUS    RESTARTS   AGE
pod/vote-5999b758fd-w6p4m   1/1     Running   0          21m
pod/vote-5999b758fd-zq9k2   1/1     Running   0          21m

NAME           TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
service/vote   NodePort   10.96.10.18   <none>        80:30300/TCP   48m

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vote   2/2     2            2           21m

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vote-5999b758fd   2         2         2       21m
```

```bash

➜  ~ kubectl get all -n prod
NAME                        READY   STATUS    RESTARTS   AGE
pod/vote-6986fd8788-5lhdf   1/1     Running   0          47m
pod/vote-6986fd8788-99lwr   1/1     Running   0          47m
pod/vote-6986fd8788-9xkz8   1/1     Running   0          47m
pod/vote-6986fd8788-ntffq   1/1     Running   0          47m

NAME           TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
service/vote   NodePort   10.96.169.49   <none>        80:30400/TCP   47m

NAME                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/vote   4/4     4            4           47m

NAME                              DESIRED   CURRENT   READY   AGE
replicaset.apps/vote-6986fd8788   4         4         4       47m
```

Access via: `http://localhost:<nodePort>`
(e.g., `http://localhost:30300`)

**If NodePort access fails**, use port-forward as an alternative.  
(In my case, NodePort failed, so I used port-forward.)

```bash
# For staging environment
kubectl port-forward svc/vote -n staging 8081:80

# For production environment
kubectl port-forward svc/vote -n prod 8082:80
```

## Environments

| **Staging**                                         | **Prod**                                            |
|-----------------------------------------------------|-----------------------------------------------------|
| [http://localhost:8081](http://localhost:8081)      | [http://localhost:8082](http://localhost:8082)      |
| <img src="https://github.com/user-attachments/assets/a0a3830a-6862-4cde-b38a-007d2bfb0882" width="400"/> | <img src="https://github.com/user-attachments/assets/582d0dad-5192-4b4b-b5af-a66411fcb50d" width="400"/> |

### 🔍 Key Learnings from this Project

* Installing and accessing ArgoCD on a local KIND Kubernetes cluster
* Setting up GitOps workflows to deploy applications to staging and production environments
* Understanding ArgoCD UI, app synchronization, and manual versus automated sync strategies
* Troubleshooting service access methods (NodePort vs. port-forwarding)
* Validating deployments visually via the ArgoCD UI and browser endpoints
