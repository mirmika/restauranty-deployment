# Running the Project Locally

## Prerequisites

- Clone the repository:
  ```bash
  git clone https://github.com/Pokfinner/restauranty.git
  cd restauranty
  ```

- Ensure Docker and Kubernetes (or Minikube) are installed.

## Dockerfile for Each Service

- Each microservice (`auth`, `discounts`, `items`, `frontend`) contains its own `Dockerfile`.
- Use `docker-compose.yml` to orchestrate services locally.
- Uses internal Docker networking for secure container communication.

## Services Overview

| Service   | Description                        | Port  |
|-----------|------------------------------------|--------|
| Auth      | Handles authentication             | 3001   |
| Discounts | Manages coupons and campaigns      | 3002   |
| Items     | Manages items and order flow       | 3003   |
| Frontend  | React-based web interface          | 3000   |
| MongoDB   | Application database               | 27017  |
| HAProxy   | Routes traffic to backend services | 80     |

## HAProxy Configuration

- Routes based on path:

  - `/api/auth` → Auth Service
  - `/api/discounts` → Discounts Service
  - `/api/items` → Items Service
  - `/` → Frontend

## Docker Deployment

To start the entire application:

```bash
docker-compose up --build
```

## Accessing the App

Visit: [http://localhost](http://localhost)

HAProxy will route:

- `/api/auth` → auth service
- `/api/discounts` → discounts service
- `/api/items` → items service
- `/` → frontend UI

# EKS Deployment

## EKS Setup

- Created an EKS cluster and configured worker nodes.

## Kubernetes Configuration

- All microservices and the frontend are configured using environment variable files stored in `k8s/config/`.

- These `.env` files are used to create Kubernetes secrets for each service:

```bash
kubectl create secret generic auth-secret --from-env-file=k8s/config/auth-config.env --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic discounts-secret --from-env-file=k8s/config/discounts-config.env --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic frontend-secret --from-env-file=k8s/config/frontend-config.env --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret generic items-secret --from-env-file=k8s/config/items-config.env --dry-run=client -o yaml | kubectl apply -f -
```

> These secrets will be mounted into pods by the respective deployment configurations for secure access to environment-specific variables.


### Backend & Frontend Manifests

- Manifests for:

  - Auth, Discounts, Items
  - Frontend (set `REACT_APP_SERVER_URL=/api`)
  - MongoDb
  - Haproxy

### HAProxy ConfigMap

```bash
kubectl create configmap haproxy-config --from-file=haproxy.cfg
```

# CI/CD with GitHub Actions

## Setup

- CI/CD via GitHub Actions for Docker builds and EKS deployment.

## CI/CD Pipelines

### Build & Push Images (on tag push)

```yaml
- Checkout Code
- Docker Login using GitHub Secrets
- Extract tag (e.g., v1.0.0)
- Build Docker Images (auth, discounts, items, frontend)
- Push to Docker Hub
```

### Deploy to EKS (on main branch push)

```yaml
- Checkout Code
- Set IMAGE_TAG
- Configure AWS Credentials via GitHub Secrets
- Install kubectl
- Update kubeconfig for EKS cluster
- Apply Kubernetes manifests with image tags
```
## Accessing Deployment via HAProxy (External IP)

After deploying to EKS, HAProxy exposes an external load balancer service. To access the application:

### Step 1: Get External IP

Run the following command to get the external IP of the HAProxy service:

```bash
kubectl get svc haproxy-service
```

Look for the `EXTERNAL-IP` in the output:

```
NAME              TYPE           CLUSTER-IP     EXTERNAL-IP       PORT(S)        AGE
haproxy-service   LoadBalancer   10.0.123.45    a1b2c3d4.elb.amazonaws.com   80:30080/TCP   5m
```

### Step 2: Access the App

Open the external IP in your browser:

```bash
http://<EXTERNAL-IP>
```

HAProxy will route based on path:

| URL Path           | Microservice     |
|--------------------|------------------|
| `/api/auth`        | Auth             |
| `/api/discounts`   | Discounts        |
| `/api/items`       | Items            |
| `/`                | React Frontend   |

Make sure your security group for the LoadBalancer allows inbound traffic on port 80 (HTTP).

# 📈 Monitoring & Logging (EKS + Prometheus + Grafana)

Monitor your EKS-based microservice stack (auth, items, discounts, frontend, HAProxy) using Prometheus and Grafana.

## ✅ Prerequisites

- EKS cluster running  
- Helm and kubectl configured  
- Update kubeconfig:
  ```bash
  aws eks update-kubeconfig --region <region> --name <cluster-name>
  ```

## 1. Add Helm Repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

## 2. Create Namespace

```bash
kubectl create namespace prometheus
```

## 3. Install Monitoring Stack

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack -n prometheus
kubectl get pods -n prometheus
```

## 4. Expose Services

Change both Prometheus and Grafana services to `LoadBalancer`:

```bash
kubectl edit svc kube-prometheus-stack-prometheus -n prometheus
kubectl edit svc kube-prometheus-stack-grafana -n prometheus
```

Set `type: LoadBalancer`

## 5. Access Grafana

- Open Grafana at `http://<EXTERNAL-IP>`
- Login:  
  - **Username:** `admin`  
  - **Password:** `prom-operator`

## 6. Create Dashboards

Use Prometheus as data source.

Example queries:
```text
container_cpu_usage_seconds_total{pod=~"haproxy.*|auth.*|discounts.*|items.*"}
container_memory_usage_bytes{namespace="default"}
kube_pod_status_ready{namespace="default"}
```

## 7. Access Prometheus

Open Prometheus at its external IP and query metrics directly.

