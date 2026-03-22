# Kubernetes Demo App

A minimal Kubernetes setup that deploys a Node.js web application backed by MongoDB, running locally via Minikube. Intended as a hands-on reference for core Kubernetes concepts: Deployments, Services, ConfigMaps, and Secrets.

reference video: https://www.youtube.com/watch?v=s_o8dwzRlu4
reference code: https://gitlab.com/nanuchi/k8s-in-1-hour/-/blob/master/webapp.yaml?ref_type=heads
---

## Architecture

```
Browser
  │
  ▼
webapp-service (NodePort :30100)
  │
  ▼
webapp-deployment  ──── mongo-service (ClusterIP :27017)
(Node.js app)               │
                            ▼
                    mongo-deployment
                      (MongoDB 5.0)
```

The web app reads the MongoDB connection URL from a ConfigMap and the credentials from a Secret, both injected as environment variables at runtime.

---

## Config Files

### `mongo-secret.yaml`
Defines a Kubernetes **Secret** named `mongo-secret` that stores the MongoDB credentials (username and password) as base64-encoded values. Secrets are the proper place for sensitive data — they are kept separate from application code and config, and Kubernetes can restrict access to them via RBAC.

The webapp and MongoDB deployments both reference this secret by key name, so credentials are never hardcoded in either manifest.

> **Note:** The values in this file are base64-encoded, not encrypted. For production use, consider a secrets management solution (e.g. Sealed Secrets, Vault, or a cloud provider's secret store).

### `mongo-config.yaml`
Defines a Kubernetes **ConfigMap** named `mongo-config` that holds non-sensitive configuration — in this case, the MongoDB service hostname (`mongo-service`). ConfigMaps decouple environment-specific settings from container images. The web app reads `mongo-url` from this ConfigMap to know where to connect.

Using the Kubernetes service name (`mongo-service`) as the URL means service discovery is handled by Kubernetes DNS automatically — no IP addresses needed.

### `mongo.yaml`
Defines two resources:

- **Deployment** (`mongo-deployment`): Runs one replica of the official `mongo:5.0` container on port `27017`. It pulls the root username and password from `mongo-secret` via environment variables (`MONGO_INITDB_ROOT_USERNAME` / `MONGO_INITDB_ROOT_PASSWORD`), which MongoDB uses to initialize the root account on first start.

- **Service** (`mongo-service`): A **ClusterIP** service that exposes MongoDB internally within the cluster on port `27017`. ClusterIP means it is only reachable by other pods inside the cluster — it is not exposed to the outside world.

### `webapp.yaml`
Defines two resources:

- **Deployment** (`webapp-deployment`): Runs one replica of the `nanajanashia/k8s-demo-app:v1.0` Node.js app on port `3000`. It receives three environment variables at runtime:
  - `DB_URL` — from the `mongo-config` ConfigMap (the MongoDB hostname)
  - `USER_NAME` — from the `mongo-secret` Secret
  - `USER_PWD` — from the `mongo-secret` Secret

- **Service** (`webapp-service`): A **NodePort** service that exposes the web app externally on port `30100`. NodePort makes the service reachable from outside the cluster by forwarding traffic from a port on the node itself down to the pod's container port (`3000`).

---

## Prerequisites

- **Docker** — Minikube uses Docker as its driver. Install from [docker.com](https://www.docker.com/products/docker-desktop/).
- **Minikube** — Local Kubernetes cluster. Install via Homebrew:
  ```bash
  brew install minikube
  ```
- **kubectl** — Kubernetes CLI. Install via Homebrew:
  ```bash
  brew install kubectl
  ```
  Or install alongside Docker Desktop, which bundles it.

---

## Usage

### 1. Start Minikube

```bash
minikube start --driver=docker
```

### 2. Apply the manifests

Order matters — the Secret and ConfigMap must exist before the deployments that reference them.

```bash
kubectl apply -f mongo-secret.yaml
kubectl apply -f mongo-config.yaml
kubectl apply -f mongo.yaml
kubectl apply -f webapp.yaml
```

Or apply all at once (Kubernetes handles dependency ordering via retries):

```bash
kubectl apply -f .
```

### 3. Verify everything is running

```bash
kubectl get all
```

You should see both pods with `STATUS: Running` and both services listed. Example output:

```
NAME                                     READY   STATUS    RESTARTS   AGE
pod/mongo-deployment-xxx                 1/1     Running   0          1m
pod/webapp-deployment-xxx                1/1     Running   0          1m

NAME                     TYPE        CLUSTER-IP     PORT(S)          AGE
service/mongo-service    ClusterIP   10.x.x.x       27017/TCP        1m
service/webapp-service   NodePort    10.x.x.x       3000:30100/TCP   1m
```

### 4. Open in the browser

On macOS, Minikube runs inside a Docker container and its internal IP (`192.168.49.2`) is not directly routable from the host. Use the `minikube service` command, which creates a tunnel and opens the app automatically:

```bash
minikube service webapp-service
```

This will print a local tunnel URL (e.g. `http://127.0.0.1:<port>`) and open it in your default browser.

> Keep the terminal running while you use the app — closing it tears down the tunnel.

**Alternative — persistent port-forward:**

If you prefer a stable local address, run this in a dedicated terminal:

```bash
kubectl port-forward svc/webapp-service 30100:3000
```

Then open `http://localhost:30100` in your browser. Keep this terminal open as well.

### 5. Tear down

```bash
kubectl delete -f .
minikube stop
```
