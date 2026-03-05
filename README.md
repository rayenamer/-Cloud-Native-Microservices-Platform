# ☁️ Cloud-Native Microservices Platform

> A production-grade, cloud-native application built on Kubernetes, featuring Java Quarkus microservices, a React frontend, a full CI/CD pipeline via GitHub Actions, NGINX Ingress with TLS, and real-time observability through Prometheus and Grafana.

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Technology Stack](#technology-stack)
3. [High-Level Architecture](#high-level-architecture)
4. [Kubernetes Internal Architecture](#kubernetes-internal-architecture)
5. [Infrastructure Setup — VMware & Terraform](#infrastructure-setup--vmware--terraform)
6. [Microservices Backend — Java Quarkus](#microservices-backend--java-quarkus)
7. [Frontend — React](#frontend--react)
8. [Containerization — Dockerfiles](#containerization--dockerfiles)
9. [Kubernetes Manifests](#kubernetes-manifests)
10. [API Gateway & Ingress Configuration](#api-gateway--ingress-configuration)
11. [CI/CD Pipeline — GitHub Actions](#cicd-pipeline--github-actions)
12. [TLS — Let's Encrypt](#tls--lets-encrypt)
13. [Monitoring — Prometheus & Grafana](#monitoring--prometheus--grafana)
14. [Build, Deploy & Test Commands](#build-deploy--test-commands)
15. [Advanced Features](#advanced-features)
16. [Environment Variables Reference](#environment-variables-reference)
17. [Project Structure](#project-structure)
18. [Troubleshooting](#troubleshooting)

---

## Project Overview

This project demonstrates a complete, end-to-end cloud-native microservices architecture designed for real-world scalability, resilience, and operational visibility. The system is self-hosted on VMware virtual machines, with Kubernetes orchestrating containerized workloads. Infrastructure is fully automated with Terraform, deployments are driven by GitHub Actions, and the platform is publicly accessible via NGINX Ingress with TLS termination.

**Goals:**
- Demonstrate enterprise-grade microservices design patterns on self-managed infrastructure.
- Automate the entire lifecycle: provision → build → test → deploy → monitor.
- Expose services securely over the internet using DNS, Ingress, and Let's Encrypt certificates.
- Provide full operational observability through metrics dashboards.

---

## Technology Stack

| Layer | Technology |
|---|---|
| **Virtualization** | VMware vSphere / Workstation |
| **Infrastructure as Code** | Terraform |
| **Container Orchestration** | Kubernetes (1 master + 2+ workers) |
| **Backend Framework** | Java 17 + Quarkus |
| **Frontend** | React 18 |
| **Containerization** | Docker |
| **Container Registry** | Docker Hub / GitHub Container Registry (GHCR) |
| **API Gateway** | Kubernetes ClusterIP + NGINX Ingress |
| **Ingress Controller** | NGINX Ingress Controller |
| **TLS** | Let's Encrypt (via cert-manager) |
| **CI/CD** | GitHub Actions |
| **Metrics** | Prometheus |
| **Dashboards** | Grafana |
| **Service Discovery** | Kubernetes DNS (CoreDNS) |
| **Inter-service Communication** | HTTP via ClusterIP Services |

---

## High-Level Architecture

```
                            ┌─────────────────────────────────────────────────────┐
                            │                   INTERNET                          │
                            └──────────────────────┬──────────────────────────────┘
                                                   │ HTTPS (443) / HTTP (80)
                                                   ▼
                            ┌──────────────────────────────────────────────────────┐
                            │           PUBLIC DNS (e.g. myapp.example.com)        │
                            └──────────────────────┬───────────────────────────────┘
                                                   │
                                                   ▼
                  ┌────────────────────────────────────────────────────────────────┐
                  │               NGINX INGRESS CONTROLLER (NodePort/LB)           │
                  │              TLS Termination via Let's Encrypt cert-manager     │
                  └──────────┬──────────────────────────────────┬──────────────────┘
                             │ /api/*                           │ /*
                             ▼                                  ▼
              ┌──────────────────────────┐       ┌──────────────────────────────┐
              │       API GATEWAY        │       │       REACT FRONTEND         │
              │   (ClusterIP Service)    │       │   (ClusterIP Service)        │
              └───────────┬──────────────┘       └──────────────────────────────┘
                          │
            ┌─────────────┼─────────────┐
            │             │             │
            ▼             ▼             ▼
  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
  │ user-service │ │order-service │ │payment-svc   │
  │  (Quarkus)   │ │  (Quarkus)   │ │  (Quarkus)   │
  │ ClusterIP    │ │ ClusterIP    │ │ ClusterIP    │
  └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
         │                │                │
         └────────────────┼────────────────┘
                          │
                          ▼
              ┌───────────────────────┐
              │      DATABASE(S)      │
              │  (PostgreSQL / MySQL) │
              │  PersistentVolume     │
              └───────────────────────┘

                          │
                          ▼
         ┌─────────────────────────────────┐
         │          MONITORING STACK        │
         │  Prometheus (scrape metrics)     │
         │  Grafana (visualize dashboards)  │
         └──────────────────────────────────┘
```

---

## Kubernetes Internal Architecture

```
┌──────────────────────────────────────────────────────────────────────────────────┐
│                         KUBERNETES CLUSTER                                       │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐    │
│  │  MASTER NODE (Control Plane)                                             │    │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────┐  ┌───────────────┐ │    │
│  │  │ API Server   │  │  Scheduler   │  │ Controller │  │    etcd       │ │    │
│  │  │ kube-apisvr  │  │              │  │  Manager   │  │  (cluster DB) │ │    │
│  │  └──────────────┘  └──────────────┘  └────────────┘  └───────────────┘ │    │
│  └─────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌──────────────────────────────────┐  ┌──────────────────────────────────────┐ │
│  │  WORKER NODE 1                   │  │  WORKER NODE 2                       │ │
│  │                                  │  │                                      │ │
│  │  ┌────────────┐ ┌─────────────┐  │  │  ┌────────────┐  ┌───────────────┐  │ │
│  │  │user-svc    │ │order-svc    │  │  │  │payment-svc │  │ react-frontend│  │ │
│  │  │Pod(s)      │ │Pod(s)       │  │  │  │Pod(s)      │  │ Pod(s)        │  │ │
│  │  └────────────┘ └─────────────┘  │  │  └────────────┘  └───────────────┘  │ │
│  │                                  │  │                                      │ │
│  │  ┌────────────────────────────┐  │  │  ┌──────────────────────────────┐   │ │
│  │  │  NGINX Ingress Controller  │  │  │  │  Prometheus + Grafana Pods   │   │ │
│  │  │  (DaemonSet / Deployment)  │  │  │  │  (monitoring namespace)      │   │ │
│  │  └────────────────────────────┘  │  │  └──────────────────────────────┘   │ │
│  │                                  │  │                                      │ │
│  │  kubelet  │  kube-proxy           │  │  kubelet  │  kube-proxy             │ │
│  └──────────────────────────────────┘  └──────────────────────────────────────┘ │
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │  CLUSTER NETWORKING (CoreDNS + kube-proxy)                                │  │
│  │  Services:  user-service.default.svc.cluster.local                        │  │
│  │             order-service.default.svc.cluster.local                       │  │
│  │             payment-service.default.svc.cluster.local                     │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────┘
```

**Namespace Layout:**

| Namespace | Workloads |
|---|---|
| `default` | user-service, order-service, payment-service, react-frontend |
| `ingress-nginx` | NGINX Ingress Controller |
| `cert-manager` | cert-manager (Let's Encrypt automation) |
| `monitoring` | Prometheus, Grafana |

---

## Infrastructure Setup — VMware & Terraform

### VM Topology

| VM | Role | vCPU | RAM | Disk |
|---|---|---|---|---|
| `k8s-master` | Control Plane | 2 | 4 GB | 40 GB |
| `k8s-worker-01` | Worker Node | 4 | 8 GB | 60 GB |
| `k8s-worker-02` | Worker Node | 4 | 8 GB | 60 GB |

### Terraform Configuration

Terraform provisions the VMware VMs and configures networking. The state is stored remotely (e.g., S3-compatible backend or Terraform Cloud).

**Directory structure:**
```
infra/
├── main.tf
├── variables.tf
├── outputs.tf
├── providers.tf
└── modules/
    ├── vm/
    └── network/
```

**`providers.tf`:**
```hcl
terraform {
  required_providers {
    vsphere = {
      source  = "hashicorp/vsphere"
      version = "~> 2.4"
    }
  }
}

provider "vsphere" {
  user                 = var.vsphere_user
  password             = var.vsphere_password
  vsphere_server       = var.vsphere_server
  allow_unverified_ssl = true
}
```

**`main.tf` (VM provisioning excerpt):**
```hcl
resource "vsphere_virtual_machine" "k8s_nodes" {
  count            = var.worker_count + 1
  name             = "k8s-node-${count.index}"
  resource_pool_id = data.vsphere_resource_pool.pool.id
  datastore_id     = data.vsphere_datastore.datastore.id
  num_cpus         = count.index == 0 ? 2 : 4
  memory           = count.index == 0 ? 4096 : 8192
  guest_id         = "ubuntu64Guest"

  network_interface {
    network_id   = data.vsphere_network.network.id
    adapter_type = "vmxnet3"
  }

  disk {
    label            = "disk0"
    size             = count.index == 0 ? 40 : 60
    eagerly_scrub    = false
    thin_provisioned = true
  }

  clone {
    template_uuid = data.vsphere_virtual_machine.template.id
    customize {
      linux_options {
        host_name = "k8s-node-${count.index}"
        domain    = "local"
      }
      network_interface {
        ipv4_address = cidrhost(var.subnet_cidr, 10 + count.index)
        ipv4_netmask = 24
      }
      ipv4_gateway = var.gateway
    }
  }
}
```

**Provision infrastructure:**
```bash
cd infra/
terraform init
terraform plan -var-file="terraform.tfvars"
terraform apply -var-file="terraform.tfvars" -auto-approve
```

### Kubernetes Cluster Bootstrap

After VMs are provisioned, the Kubernetes cluster is bootstrapped using `kubeadm`:

**On the master node:**
```bash
# Install containerd, kubelet, kubeadm, kubectl
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
sudo systemctl enable kubelet

# Initialize the cluster
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --apiserver-advertise-address=<MASTER_IP>

# Configure kubectl for the current user
mkdir -p $HOME/.kube
sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

# Install Flannel CNI
kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
```

**On each worker node:**
```bash
sudo kubeadm join <MASTER_IP>:6443 \
  --token <TOKEN> \
  --discovery-token-ca-cert-hash sha256:<HASH>
```

**Verify cluster:**
```bash
kubectl get nodes
# NAME             STATUS   ROLES           AGE   VERSION
# k8s-master       Ready    control-plane   5m    v1.29.x
# k8s-worker-01    Ready    <none>          3m    v1.29.x
# k8s-worker-02    Ready    <none>          3m    v1.29.x
```

---

## Microservices Backend — Java Quarkus

### Service Descriptions

| Service | Responsibilities | Port |
|---|---|---|
| `user-service` | User registration, authentication, profile management | 8080 |
| `order-service` | Order creation, status tracking, order history | 8081 |
| `payment-service` | Payment processing, payment status, transaction records | 8082 |

### Inter-Service Communication

Services communicate via Kubernetes DNS using ClusterIP service names:

```
http://user-service:8080/api/users/{id}
http://order-service:8081/api/orders/{id}
http://payment-service:8082/api/payments/{id}
```

No service mesh is required for this setup; DNS resolution is handled by CoreDNS within the cluster.

### Example REST Endpoint (Quarkus)

```java
// user-service/src/main/java/com/example/user/UserResource.java
@Path("/api/users")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class UserResource {

    @Inject
    UserService userService;

    @GET
    @Path("/{id}")
    public Response getUser(@PathParam("id") Long id) {
        return userService.findById(id)
            .map(user -> Response.ok(user).build())
            .orElse(Response.status(Response.Status.NOT_FOUND).build());
    }

    @POST
    public Response createUser(UserDTO userDTO) {
        User user = userService.create(userDTO);
        return Response.status(Response.Status.CREATED).entity(user).build();
    }
}
```

### application.properties (Quarkus)

```properties
# user-service/src/main/resources/application.properties
quarkus.http.port=8080
quarkus.datasource.db-kind=postgresql
quarkus.datasource.username=${DB_USER:postgres}
quarkus.datasource.password=${DB_PASSWORD:secret}
quarkus.datasource.jdbc.url=jdbc:postgresql://${DB_HOST:localhost}:5432/${DB_NAME:users_db}
quarkus.hibernate-orm.database.generation=update

# Kubernetes health checks
quarkus.smallrye-health.root-path=/q/health

# Micrometer metrics for Prometheus
quarkus.micrometer.export.prometheus.enabled=true
quarkus.micrometer.export.prometheus.path=/q/metrics
```

---

## Frontend — React

The React frontend is a single-page application (SPA) that communicates exclusively with the API Gateway, which routes requests to the appropriate backend services.

```
src/
├── components/
│   ├── Users/
│   ├── Orders/
│   └── Payments/
├── services/
│   └── api.js          # Axios API client pointing to /api/*
├── App.jsx
└── index.js
```

**`api.js` — API client:**
```javascript
import axios from 'axios';

const api = axios.create({
  baseURL: process.env.REACT_APP_API_BASE_URL || '/api',
  headers: { 'Content-Type': 'application/json' }
});

export const getUser = (id) => api.get(`/users/${id}`);
export const createOrder = (order) => api.post('/orders', order);
export const processPayment = (payment) => api.post('/payments', payment);

export default api;
```

The frontend uses relative `/api` paths that are resolved by the NGINX Ingress routing rules — no hardcoded service URLs.

---

## Containerization — Dockerfiles

### Quarkus Service Dockerfile (Multi-Stage Build)

```dockerfile
# user-service/Dockerfile
# Stage 1: Build
FROM eclipse-temurin:17-jdk AS build
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests -Dquarkus.package.type=uber-jar

# Stage 2: Runtime (minimal image)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*-runner.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

> **Note:** For Quarkus native builds using GraalVM (smaller, faster startup):
> ```dockerfile
> FROM quay.io/quarkus/ubi-quarkus-mandrel-builder-image:23.1 AS build
> WORKDIR /app
> COPY . .
> RUN ./mvnw package -Pnative -DskipTests
>
> FROM registry.access.redhat.com/ubi8/ubi-minimal:8.8
> WORKDIR /app
> COPY --from=build /app/target/*-runner ./application
> EXPOSE 8080
> ENTRYPOINT ["./application"]
> ```

### React Frontend Dockerfile (Multi-Stage Build)

```dockerfile
# frontend/Dockerfile
# Stage 1: Build
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: Serve with NGINX
FROM nginx:1.25-alpine
COPY --from=build /app/build /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

**`nginx.conf` for React SPA (handles client-side routing):**
```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /health {
        return 200 'ok';
        add_header Content-Type text/plain;
    }
}
```

### Build Docker Images Locally

```bash
# Build user-service
docker build -t user-service:latest ./user-service

# Build order-service
docker build -t order-service:latest ./order-service

# Build payment-service
docker build -t payment-service:latest ./payment-service

# Build React frontend
docker build -t react-frontend:latest ./frontend
```

---

## Kubernetes Manifests

All Kubernetes manifests live in a dedicated `k8s/` repository (or directory). This separation enables the CI/CD pipeline to manage infrastructure independently from application code.

### Deployment — user-service

```yaml
# k8s/deployments/user-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
  namespace: default
  labels:
    app: user-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: user-service
  template:
    metadata:
      labels:
        app: user-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/q/metrics"
        prometheus.io/port: "8080"
    spec:
      containers:
        - name: user-service
          image: ghcr.io/<your-org>/user-service:latest
          ports:
            - containerPort: 8080
          env:
            - name: DB_HOST
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: host
            - name: DB_USER
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: password
            - name: DB_NAME
              value: users_db
          readinessProbe:
            httpGet:
              path: /q/health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /q/health/live
              port: 8080
            initialDelaySeconds: 20
            periodSeconds: 10
          resources:
            requests:
              cpu: "250m"
              memory: "256Mi"
            limits:
              cpu: "500m"
              memory: "512Mi"
```

### ClusterIP Service — user-service

```yaml
# k8s/services/user-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: user-service
  namespace: default
spec:
  selector:
    app: user-service
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
  type: ClusterIP
```

The same pattern applies to `order-service` (port 8081) and `payment-service` (port 8082).

### Deployment — React Frontend

```yaml
# k8s/deployments/react-frontend.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: react-frontend
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: react-frontend
  template:
    metadata:
      labels:
        app: react-frontend
    spec:
      containers:
        - name: react-frontend
          image: ghcr.io/<your-org>/react-frontend:latest
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /health
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
```

### Secret — Database Credentials

```yaml
# k8s/secrets/db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
  namespace: default
type: Opaque
stringData:
  host: "postgres-service"
  username: "appuser"
  password: "supersecret"
  database: "appdb"
```

> ⚠️ **Never commit real secrets to Git.** Use [Sealed Secrets](https://sealed-secrets.netlify.app/), [Vault](https://www.vaultproject.io/), or environment-specific overlays with Kustomize.

---

## API Gateway & Ingress Configuration

### NGINX Ingress Controller Installation

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.9.4/deploy/static/provider/cloud/deploy.yaml
```

### Ingress Resource

```yaml
# k8s/ingress/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: tls-secret
  rules:
    - host: myapp.example.com
      http:
        paths:
          # Backend API routing
          - path: /api/users(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 8080
          - path: /api/orders(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: order-service
                port:
                  number: 8081
          - path: /api/payments(/|$)(.*)
            pathType: Prefix
            backend:
              service:
                name: payment-service
                port:
                  number: 8082
          # Frontend — catch-all
          - path: /
            pathType: Prefix
            backend:
              service:
                name: react-frontend
                port:
                  number: 80
```

**Routing summary:**

| Path Pattern | Routes To |
|---|---|
| `myapp.example.com/api/users/*` | `user-service:8080` |
| `myapp.example.com/api/orders/*` | `order-service:8081` |
| `myapp.example.com/api/payments/*` | `payment-service:8082` |
| `myapp.example.com/*` | `react-frontend:80` |

---

## TLS — Let's Encrypt

### cert-manager Installation

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml
```

### ClusterIssuer (Let's Encrypt Production)

```yaml
# k8s/cert-manager/cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            class: nginx
```

cert-manager will automatically provision and renew TLS certificates. The certificate is stored in the `tls-secret` Kubernetes Secret referenced by the Ingress.

---

## CI/CD Pipeline — GitHub Actions

### Repository Structure

```
github-org/
├── app-repo/               # Application source code (all services + frontend)
│   └── .github/workflows/
│       ├── user-service-ci.yml
│       ├── order-service-ci.yml
│       ├── payment-service-ci.yml
│       └── frontend-ci.yml
│
└── k8s-repo/               # Kubernetes manifests only
    └── k8s/
        ├── deployments/
        ├── services/
        ├── ingress/
        └── secrets/
```

### CI Workflow — Build & Push (user-service)

```yaml
# .github/workflows/user-service-ci.yml
name: CI — user-service

on:
  push:
    branches: [main]
    paths:
      - 'user-service/**'

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build with Maven
        run: |
          cd user-service
          ./mvnw package -DskipTests

      - name: Log in to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./user-service
          push: true
          tags: |
            ghcr.io/${{ github.repository_owner }}/user-service:latest
            ghcr.io/${{ github.repository_owner }}/user-service:${{ github.sha }}
```

### CD Workflow — Deploy to Kubernetes

```yaml
# .github/workflows/deploy.yml
name: CD — Deploy to Kubernetes

on:
  workflow_run:
    workflows: ["CI — user-service", "CI — order-service", "CI — payment-service", "CI — frontend"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    if: ${{ github.event.workflow_run.conclusion == 'success' }}

    steps:
      - name: Checkout k8s-repo
        uses: actions/checkout@v4
        with:
          repository: your-org/k8s-repo
          token: ${{ secrets.K8S_REPO_TOKEN }}

      - name: Set up kubectl
        uses: azure/setup-kubectl@v3
        with:
          version: 'v1.29.0'

      - name: Configure kubeconfig
        run: |
          mkdir -p $HOME/.kube
          echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > $HOME/.kube/config
          chmod 600 $HOME/.kube/config

      - name: Update image tags
        run: |
          kubectl set image deployment/user-service \
            user-service=ghcr.io/${{ github.repository_owner }}/user-service:${{ github.sha }} \
            -n default

      - name: Apply updated manifests
        run: |
          kubectl apply -f k8s/deployments/ -n default
          kubectl apply -f k8s/services/ -n default
          kubectl apply -f k8s/ingress/ -n default

      - name: Wait for rollout
        run: |
          kubectl rollout status deployment/user-service -n default --timeout=120s
          kubectl rollout status deployment/order-service -n default --timeout=120s
          kubectl rollout status deployment/payment-service -n default --timeout=120s
          kubectl rollout status deployment/react-frontend -n default --timeout=120s
```

**Required GitHub Secrets:**

| Secret | Description |
|---|---|
| `GITHUB_TOKEN` | Auto-provided for GHCR push |
| `K8S_REPO_TOKEN` | PAT with access to k8s-repo |
| `KUBECONFIG_B64` | Base64-encoded kubeconfig of the cluster |

---

## Monitoring — Prometheus & Grafana

### Install Prometheus Stack via Helm

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  --set grafana.adminPassword=admin \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false
```

### ServiceMonitor for Quarkus Services

Quarkus exposes metrics at `/q/metrics` via Micrometer. A `ServiceMonitor` tells Prometheus to scrape them:

```yaml
# k8s/monitoring/service-monitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: microservices-monitor
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      monitoring: "true"
  endpoints:
    - port: http
      path: /q/metrics
      interval: 15s
```

Add the label `monitoring: "true"` to your ClusterIP services to enable scraping.

### Key Metrics Tracked

| Metric | Description |
|---|---|
| `container_cpu_usage_seconds_total` | CPU usage per pod/container |
| `container_memory_working_set_bytes` | Memory usage per pod |
| `http_server_requests_seconds` | HTTP request duration (Quarkus) |
| `kube_pod_status_phase` | Pod health status |
| `kube_deployment_status_replicas_available` | Available replicas per deployment |

### Accessing Grafana

```bash
# Port-forward Grafana locally
kubectl port-forward svc/kube-prometheus-stack-grafana 3000:80 -n monitoring

# Open in browser:
# http://localhost:3000
# Default credentials: admin / admin
```

Import the **Kubernetes Cluster Monitoring** dashboard (ID: `315`) and **JVM Micrometer** dashboard (ID: `4701`) from Grafana's public dashboard library.

---

## Build, Deploy & Test Commands

### Local Development

```bash
# Start a Quarkus service in dev mode (hot reload)
cd user-service
./mvnw quarkus:dev

# Start React frontend
cd frontend
npm install
npm start
```

### Docker Compose (local integration testing)

```bash
docker-compose up --build
# Starts all services + PostgreSQL locally
```

### Apply All Kubernetes Manifests

```bash
# Deploy all services
kubectl apply -f k8s/secrets/
kubectl apply -f k8s/deployments/
kubectl apply -f k8s/services/
kubectl apply -f k8s/ingress/

# Check rollout status
kubectl rollout status deployment --all -n default
```

### Inspect Running Workloads

```bash
# View all pods
kubectl get pods -n default

# View pod logs
kubectl logs -f deployment/user-service -n default

# Describe a pod for events/errors
kubectl describe pod <pod-name> -n default

# View all services
kubectl get services -n default

# View ingress
kubectl get ingress -n default
```

### Test the API

```bash
# Test user endpoint (replace with your domain)
curl -s https://myapp.example.com/api/users/1 | jq .

# Test order creation
curl -s -X POST https://myapp.example.com/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "items": [{"productId": 42, "quantity": 2}]}' | jq .

# Test payment processing
curl -s -X POST https://myapp.example.com/api/payments \
  -H "Content-Type: application/json" \
  -d '{"orderId": 100, "amount": 49.99, "currency": "USD"}' | jq .
```

### Rolling Update (manual)

```bash
# Update a service image manually
kubectl set image deployment/user-service \
  user-service=ghcr.io/your-org/user-service:v1.2.0 \
  -n default

# Monitor rollout
kubectl rollout status deployment/user-service -n default

# Roll back if needed
kubectl rollout undo deployment/user-service -n default
```

---

## Advanced Features

### Horizontal Pod Autoscaler (HPA)

Automatically scales pods based on CPU usage:

```yaml
# k8s/hpa/user-service-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: user-service-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: user-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

```bash
kubectl apply -f k8s/hpa/
kubectl get hpa -n default
```

### ConfigMaps for Environment Configuration

```yaml
# k8s/configmaps/app-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  APP_ENV: production
  LOG_LEVEL: INFO
  MAX_POOL_SIZE: "10"
```

Reference in Deployment:
```yaml
envFrom:
  - configMapRef:
      name: app-config
```

### Network Policies (Pod-Level Firewall)

Restrict traffic so only the API gateway can reach backend services:

```yaml
# k8s/network-policies/backend-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-ingress-to-services
  namespace: default
spec:
  podSelector:
    matchLabels:
      tier: backend
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: nginx-ingress
      ports:
        - port: 8080
        - port: 8081
        - port: 8082
```

### Pod Disruption Budgets (PDB)

Ensures minimum availability during node drain or rolling updates:

```yaml
# k8s/pdb/user-service-pdb.yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: user-service-pdb
  namespace: default
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: user-service
```

---

## Environment Variables Reference

### Backend Services (Quarkus)

| Variable | Description | Default |
|---|---|---|
| `DB_HOST` | Database hostname | `localhost` |
| `DB_PORT` | Database port | `5432` |
| `DB_NAME` | Database name | `appdb` |
| `DB_USER` | Database username | `postgres` |
| `DB_PASSWORD` | Database password | *(required)* |
| `QUARKUS_HTTP_PORT` | Service HTTP port | `8080` |
| `LOG_LEVEL` | Logging level | `INFO` |

### Frontend (React)

| Variable | Description | Default |
|---|---|---|
| `REACT_APP_API_BASE_URL` | Backend API base URL | `/api` |
| `REACT_APP_ENV` | Environment name | `production` |

---

## Project Structure

```
.
├── user-service/               # Quarkus microservice — users
│   ├── src/main/java/
│   ├── src/main/resources/
│   ├── Dockerfile
│   └── pom.xml
│
├── order-service/              # Quarkus microservice — orders
│   ├── src/main/java/
│   ├── src/main/resources/
│   ├── Dockerfile
│   └── pom.xml
│
├── payment-service/            # Quarkus microservice — payments
│   ├── src/main/java/
│   ├── src/main/resources/
│   ├── Dockerfile
│   └── pom.xml
│
├── frontend/                   # React SPA
│   ├── src/
│   ├── public/
│   ├── nginx.conf
│   ├── Dockerfile
│   └── package.json
│
├── infra/                      # Terraform infrastructure
│   ├── main.tf
│   ├── variables.tf
│   ├── outputs.tf
│   └── providers.tf
│
├── k8s/                        # Kubernetes manifests
│   ├── deployments/
│   ├── services/
│   ├── ingress/
│   ├── secrets/
│   ├── configmaps/
│   ├── hpa/
│   ├── pdb/
│   ├── network-policies/
│   └── monitoring/
│
├── .github/
│   └── workflows/
│       ├── user-service-ci.yml
│       ├── order-service-ci.yml
│       ├── payment-service-ci.yml
│       ├── frontend-ci.yml
│       └── deploy.yml
│
└── docker-compose.yml          # Local development stack
```

---

## Troubleshooting

**Pods stuck in `Pending` state:**
```bash
kubectl describe pod <pod-name> -n default
# Look for: Insufficient CPU/memory, no available nodes, or unbound PVC
kubectl get events -n default --sort-by=.lastTimestamp
```

**Ingress not routing traffic:**
```bash
kubectl describe ingress app-ingress -n default
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller
```

**TLS certificate not issuing:**
```bash
kubectl describe certificate tls-secret -n default
kubectl describe certificaterequest -n default
kubectl logs -n cert-manager deployment/cert-manager
```

**CI/CD deploy failing:**
- Confirm `KUBECONFIG_B64` secret is valid: `echo "<secret>" | base64 -d | kubectl --kubeconfig /dev/stdin get nodes`
- Ensure the GitHub Actions runner can reach the cluster API server (firewall rules / VPN).

**Metrics not appearing in Prometheus:**
```bash
kubectl port-forward svc/kube-prometheus-stack-prometheus 9090:9090 -n monitoring
# Open http://localhost:9090/targets and verify your services appear as UP
```

---

## License

MIT License — see [LICENSE](LICENSE) for details.

---

*Built with ❤️ using Java Quarkus, React, Kubernetes, Terraform, and GitHub Actions.*
