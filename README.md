# CICD using ArgoCD, Git, Logstash Pipelines

This guide helps you set up a CI/CD pipeline using ArgoCD, Git, and Logstash pipelines, running on Kubernetes with Podman Desktop and `kubectl`.

---

## Prerequisites

- [Podman Desktop](https://podman-desktop.io/) (with Kubernetes enabled)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)
- [ArgoCD CLI](https://argo-cd.readthedocs.io/en/stable/cli_installation/)
- [Git](https://git-scm.com/)
- [Logstash](https://www.elastic.co/logstash)

---

## Setup Instructions

### 1. Start Kubernetes with Podman Desktop

- Open Podman Desktop.
- Enable Kubernetes in the settings.
- Wait for the cluster to be ready.

### 2. Install ArgoCD

```sh
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

#### Expose ArgoCD API Server (for local testing)

```sh
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### 3. Install ArgoCD CLI

Follow [official instructions](https://argo-cd.readthedocs.io/en/stable/cli_installation/).

### 4. Get ArgoCD Admin Password

```sh
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

### 5. Login to ArgoCD

```sh
argocd login localhost:8080 --username admin --password <password>
```

### 6. Clone Sample Project

```sh
git clone https://github.com/your-org/sample-argocd-app.git
cd sample-argocd-app
```

### 7. Create a Sample Kubernetes App

Sample directory structure:

```
sample-argocd-app/
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── logstash/
│   └── pipeline.conf
├── .git/
└── README.md
```

#### Example: `k8s/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: sample-app
          image: nginx:alpine
          ports:
            - containerPort: 80
```

#### Example: `k8s/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
spec:
  selector:
    app: sample-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: ClusterIP
```

#### Example: `logstash/pipeline.conf`

```conf
input { stdin { } }
output { stdout { codec => rubydebug } }
```

---

## Deploy with ArgoCD

1. Push your repo to GitHub/GitLab.
2. In ArgoCD UI, create a new App:
   - Repo URL: your repo
   - Path: `k8s`
   - Cluster: in-cluster
   - Namespace: default

ArgoCD will sync and deploy your app.

---

## Running Logstash

You can run Logstash as a Pod or locally:

```sh
logstash -f logstash/pipeline.conf
```

Or create a Kubernetes deployment for Logstash.

---

## Setting Up Logstash in Kubernetes

1. **Ensure you have these files in your `k8s/` directory:**
   - `logstash-configmap.yaml`
   - `logstash-deployment.yaml`
   - (Optional) `logstash-service.yaml` if you want to expose Logstash

2. **Apply the manifests:**
   ```sh
   kubectl apply -f k8s/logstash-configmap.yaml
   kubectl apply -f k8s/logstash-deployment.yaml
   kubectl apply -f k8s/logstash-service.yaml   # if needed
   ```

3. **Verify Logstash is running:**
   ```sh
   kubectl get pods -l app=logstash
   kubectl logs deployment/logstash
   ```

4. **To update the Logstash config:**
   - Edit `logstash/pipeline.conf`
   - Update `k8s/logstash-configmap.yaml` with the new contents
   - Commit and push changes
   - ArgoCD will redeploy Logstash automatically

---

## Deploying Logstash with ArgoCD

1. **Create Logstash ConfigMap and Deployment:**

   - Ensure you have `k8s/logstash-configmap.yaml` and `k8s/logstash-deployment.yaml` in your repo.
   - The ConfigMap should contain your `pipeline.conf` contents.
   - The Deployment should mount the ConfigMap as described in this repo.

2. **Apply Manifests (if testing manually):**

   ```sh
   kubectl apply -f k8s/logstash-configmap.yaml
   kubectl apply -f k8s/logstash-deployment.yaml
   ```

3. **ArgoCD Setup:**

   - Make sure your ArgoCD application includes the `k8s/` folder.
   - When you push changes to `logstash/pipeline.conf` (and update the ConfigMap YAML), ArgoCD will detect the change and redeploy Logstash automatically.

---

## Testing ArgoCD Logstash Config Updates

1. **Edit `logstash/pipeline.conf`** with a new input/output or message.
2. **Update `k8s/logstash-configmap.yaml`** with the new contents under `pipeline.conf: |`.
3. **Commit and push** your changes to Git.
4. **Watch ArgoCD UI** or run:

   ```sh
   kubectl get pods
   ```

   to see Logstash pod restarting.
5. **Check logs** to verify Logstash is running with the new config:

   ```sh
   kubectl logs deployment/logstash
   ```

---

## Accessing Logstash and Checking Logs

### Access Logstash Pod

1. **List Logstash pods:**
   ```sh
   kubectl get pods -l app=logstash
   ```

2. **Get logs from the Logstash pod:**
   ```sh
   kubectl logs deployment/logstash
   ```
   Or, if you want logs from a specific pod:
   ```sh
   kubectl logs <logstash-pod-name>
   ```

3. **(Optional) Exec into the Logstash pod for troubleshooting:**
   ```sh
   kubectl exec -it <logstash-pod-name> -- /bin/bash
   ```

### Access Logstash Service (if exposed)

If you have deployed `logstash-service.yaml` and want to access Logstash from outside the cluster:

- **Port-forward the service:**
  ```sh
  kubectl port-forward svc/logstash 5044:5044
  ```
  Then connect to `localhost:5044` from your local machine.

---

## Useful Commands

- `kubectl get pods -A`
- `kubectl logs <pod>`

---

## References

- [ArgoCD Docs](https://argo-cd.readthedocs.io/)
- [Logstash Docs](https://www.elastic.co/guide/en/logstash/current/index.html)
- [Podman Desktop](https://podman-desktop.io/)