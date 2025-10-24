# Complete EKS GitOps Setup with ArgoCD, Argo Rollouts, and Monitoring Stack

This comprehensive guide walks you through setting up a complete GitOps platform on EKS with ArgoCD, Argo Rollouts for Blue/Green deployments, and a full monitoring stack including Prometheus, Grafana, and Loki.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Helm Installation](#helm-installation)
3. [ArgoCD Installation and Setup](#argocd-installation-and-setup)
4. [Application Deployment with GitOps](#application-deployment-with-gitops)
5. [Monitoring Stack Setup](#monitoring-stack-setup)
6. [Argo Rollouts Blue/Green Deployment](#argo-rollouts-bluegreen-deployment)
7. [Complete Architecture](#complete-architecture)
8. [Kubernetes and EKS Glossary](#kubernetes-and-eks-glossary)

## Prerequisites

- EKS cluster with kubectl configured
- Git repository with production and staging branches
- Docker images ready for deployment

## Helm Installation

### Step 1: Install Helm

```bash
# Download and install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Verify Helm installation
helm version
```

### Step 2: Add Required Helm Repositories

```bash
# Add Prometheus community repository (for monitoring stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

# Add Grafana repository (for Loki)
helm repo add grafana https://grafana.github.io/helm-charts

# Update repository cache
helm repo update
```

## ArgoCD Installation and Setup

### Step 1: Install ArgoCD

```bash
# Create ArgoCD namespace
kubectl create namespace argocd

# Install ArgoCD components
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Step 2: Verify Installation

```bash
# Check if all pods are running
kubectl get pods -n argocd
```

### Step 3: Expose ArgoCD Server

```bash
# Change service type to LoadBalancer for external access
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Get the external IP address
kubectl get svc argocd-server -n argocd -o=jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

### Step 4: Get Initial Admin Password

```bash
# Retrieve the initial admin password
argocd admin initial-password -n argocd
```

**Default Credentials:**
- Username: `admin`
- Password: `` (password will be retrieved from the command above)

### Step 5: Access ArgoCD

**Option A: Port Forward (Local Access)**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# Access at: https://localhost:8080
```

**Option B: LoadBalancer (External Access)**
```bash
# Login using the external IP
argocd login <Endpoint>
```

## Application Deployment with GitOps

### Step 1: Repository Setup

Create GitHub repository with two branches:
- `prod` - Production environment
- `staging` - Staging environment

**Required Files in Each Branch:**
- `namespace.yaml` - Kubernetes namespace definition
- `service.yaml` - Service configuration
- `deployment.yaml` - Application deployment

### Step 2: Create Namespaces

```bash
# Create namespaces for applications
kubectl create namespace staging
kubectl create namespace production
```

### Step 3: Deploy Applications via ArgoCD Web UI

1. **Access ArgoCD Web Interface**
   - Navigate to the LoadBalancer IP or localhost:8080
   - Login with admin credentials

2. **Create Production Application**
   - Click "New App"
   - Application Name: `prod.utility`
   - Project: `default`
   - Sync Policy: `Automatic`
   - Repository URL: Your GitHub repository
   - Path: Root directory
   - Cluster: `https://kubernetes.default.svc`
   - Namespace: `production`

3. **Create Staging Application**
   - Click "New App"
   - Application Name: `staging.utility`
   - Project: `default`
   - Sync Policy: `Automatic`
   - Repository URL: Your GitHub repository
   - Path: Root directory
   - Cluster: `https://kubernetes.default.svc`
   - Namespace: `staging`

### Step 4: Verify Application Deployment

```bash
# Check application status
argocd app get prod.utility
argocd app get staging.utility

# Verify services are running
kubectl get svc -n staging
kubectl get svc -n production
```

## Monitoring Stack Setup

### Step 1: Verify Helm Repositories

```bash
# Verify repositories are already added (from Helm Installation section)
helm repo list
```

### Step 2: Install Monitoring Stack

```bash
# Create monitoring namespace
kubectl create namespace monitoring

# Install kube-prometheus-stack (includes Prometheus + Grafana)
helm install prometheus-stack prometheus-community/kube-prometheus-stack --namespace monitoring
```

### Step 3: Verify Installation

```bash
# Check deployment status
kubectl --namespace monitoring get pods -l "release=prometheus-stack"

# Get Grafana admin password
kubectl --namespace monitoring get secrets prometheus-stack-grafana -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

### Step 4: Access Monitoring Tools

**Access Grafana:**
```bash
# Port forward to access Grafana
kubectl port-forward -n monitoring svc/prometheus-stack-grafana 3000:80
# Access at: http://localhost:3000
```

**Access Prometheus:**
```bash
# Port forward to access Prometheus
kubectl port-forward -n monitoring svc/prometheus-stack-kube-prom-prometheus 9090:9090 &
# Access at: http://localhost:9090
```

### Step 5: Loki Logging Setup

```bash
# Install Loki stack (disable Grafana and Prometheus as they're already installed)
helm install loki-stack grafana/loki-stack --namespace monitoring --set grafana.enabled=false --set prometheus.enabled=false

# Check Loki pods
kubectl get pods -n monitoring

# Get Loki service details
kubectl get svc -n monitoring
```

### Step 6: Configure Loki as Data Source

1. **Access Grafana** (http://localhost:3000)
2. **Navigate to Configuration > Data Sources**
3. **Add Loki Data Source**
4. **Configure Loki URL** using the private IP and port from the service
5. **Test and Save** the data source configuration

## Argo Rollouts Blue/Green Deployment

### Step 1: Install Argo Rollouts Controller

```bash
# Create dedicated namespace for rollouts controller
kubectl create namespace argo-rollouts

# Download and install all Argo Rollouts components (controller, CRDs, RBAC)
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Download the kubectl plugin for rollouts management
curl -LO https://github.com/argoproj/argo-rollouts/releases/latest/download/kubectl-argo-rollouts-linux-amd64

# Install plugin so you can use 'kubectl argo rollouts' commands
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts
```

### Step 2: Update Production Branch Files

Switch to your production branch:

```bash
git checkout prod
```

#### File 1: namespace.yaml

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    environment: production
```

**Purpose:** Defines the Kubernetes namespace where your app runs.

#### File 2: rollout.yaml (Replace deployment.yaml)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: utility-rollout
  namespace: production
  labels:
    app: utility-app
    environment: production
spec:
  replicas: 3
  strategy:
    blueGreen:
      activeService: utility-service
      previewService: utility-preview-service
      autoPromotionEnabled: false    # Manual promotion for production safety
      scaleDownDelaySeconds: 300     # 5 minutes to validate before scaling down old version
      prePromotionAnalysis:
        templates:
        - templateName: health-check
        args:
        - name: service-name
          value: utility-preview-service
  selector:
    matchLabels:
      app: utility-app
  template:
    metadata:
      labels:
        app: utility-app
        environment: production
    spec:
      containers:
      - name: utility-container
        image: noelav07/utility:3.0    # Current production image
        ports:
        - containerPort: 8090
        env:
        - name: ENVIRONMENT
          value: "production"
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        readinessProbe:
          httpGet:
            path: /
            port: 8090
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /
            port: 8090
          initialDelaySeconds: 30
          periodSeconds: 30
```

**Configuration Explanation:**
- `activeService`: Service routing production traffic to current (Blue) version
- `previewService`: Service routing test traffic to new (Green) version
- `autoPromotionEnabled: false`: You control when to switch traffic (not automatic)
- `scaleDownDelaySeconds: 300`: After promotion, wait 5 minutes before removing old version

#### File 3: utility-service.yaml (Active service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: utility-service
  namespace: production
  labels:
    app: utility-app
    environment: production
spec:
  selector:
    app: utility-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8090
  type: LoadBalancer
```

**Purpose:** Your existing service becomes the "active" service. Production traffic flows here.

#### File 4: utility-preview-service.yaml (Preview service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: utility-preview-service
  namespace: production
  labels:
    app: utility-app
    environment: production
    service-type: preview
spec:
  selector:
    app: utility-app
  ports:
  - name: http
    protocol: TCP
    port: 80
    targetPort: 8090
  type: LoadBalancer    # To access and test Green version
```

**Purpose:** New service that lets you test the Green version before promoting it.

#### File 5: analysis-template.yaml (Health check for rollouts)

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: health-check
  namespace: production
spec:
  args:
  - name: service-name
  metrics:
  - name: webmetric
    successCondition: result == "true"
    provider:
      web:
        url: "http://{{args.service-name}}.production.svc.cluster.local/"
        timeoutSeconds: 20
        headers:
        - key: Host
          value: "example.com"
        jsonPath: "{$.status}"
```

**Purpose:** Automated health check that verifies new version is responding before allowing promotion.

### Step 3: Commit and Deploy Changes

```bash
# Backup old deployment file
mv deployment.yaml deployment.yaml.bak

# Add new files to git
git add namespace.yaml rollout.yaml utility-service.yaml utility-preview-service.yaml analysis-template.yaml

# Commit changes
git commit -m "Convert to Argo Rollouts with Blue/Green deployment strategy"

# Push to trigger ArgoCD sync
git push origin prod
```

### Step 4: Trigger Blue/Green Deployment

#### Step 4.1: Edit the image tag in rollout.yaml

Change from:
```yaml
image: noelav07/utility:3.0
```

To:
```yaml
image: noelav07/utility:1.0
```

#### Step 4.2: Commit and push

```bash
git add rollout.yaml
git commit -m "Blue/Green rollout: utility:3.0 → utility:1.0"
git push origin prod
```

#### What happens when you push:

1. ArgoCD detects the image change
2. Argo Rollouts starts creating Green version alongside Blue
3. Blue (3.0) continues serving production traffic
4. Green (1.0) gets deployed but only accessible via preview service
5. You get to test Green before switching traffic

### Step 5: Monitor and Test

#### Monitor the Rollout

```bash
# Watch rollout progress (live updates)
kubectl argo rollouts get rollout utility-rollout -n production --watch

# Check services
kubectl get svc -n production

# Get external IPs for testing
kubectl get svc utility-service utility-preview-service -n production -o wide
```

#### Expected Rollout Status

```
Name:            utility-rollout
Namespace:       production  
Status:          ॥ Paused
Strategy:        BlueGreen
Replicas:        3
Revision:        2
```

**Status Meanings:**
- `Progressing`: Green version is being created
- `Paused`: Green is ready, waiting for your promotion
- `Healthy`: Everything is running correctly

#### Test Green Version

```bash
# Get preview service URL
PREVIEW_URL=$(kubectl get svc utility-preview-service -n production -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Test Green version at: http://$PREVIEW_URL"

# Test the new version
curl http://$PREVIEW_URL
```

**Why this is important:**
- Green version is running in production environment
- Same resources, same network as production
- But isolated traffic - only you can access it
- Validates new version before real users see it

### Step 6: Promote or Abort

#### If Green version looks good, promote it:

```bash
kubectl argo rollouts promote utility-rollout -n production
```

**What happens:**
- Traffic switches from Blue to Green instantly
- Users now see version 1.0 instead of 3.0
- Blue version stays running for 5 minutes (scaleDownDelaySeconds)
- After 5 minutes, Blue version gets removed
- Green becomes the new Blue for next rollout

#### If there are issues, abort and rollback:

```bash
kubectl argo rollouts abort utility-rollout -n production
```

**What happens:**
- Green version gets removed immediately
- Blue version continues serving production traffic
- No impact to users - they never saw the broken version
- Rollout status shows "Degraded" or "Failed"

### Step 7: Monitoring Commands

```bash
# Check rollout status
kubectl argo rollouts get rollout utility-rollout -n production

# List all rollouts
kubectl argo rollouts list rollouts -n production

# Get rollout history
kubectl argo rollouts history utility-rollout -n production

# Watch live rollout
kubectl argo rollouts get rollout utility-rollout -n production --watch

# Check both services
kubectl get svc -n production
```

## Complete Architecture

### Final Setup Summary

**1. ArgoCD GitOps Setup**
- ✅ ArgoCD installed in EKS cluster
- ✅ Exposed via LoadBalancer with public endpoint
- ✅ Created GitHub repo with prod and staging branches
- ✅ Deployed applications via ArgoCD Web UI pulling from respective branches
- ✅ Multi-environment deployment working perfectly

**2. Complete Monitoring Stack**
- ✅ Prometheus + Grafana via kube-prometheus-stack (one-command install!)
- ✅ Loki logging via loki-stack integration
- ✅ All pre-configured dashboards and data sources
- ✅ Full observability for metrics, dashboards, and logs

**3. Application Infrastructure**
- ✅ Production environment (production namespace) with LoadBalancer
- ✅ Staging environment (staging namespace) with LoadBalancer
- ✅ Public endpoints for both applications
- ✅ Namespace isolation and proper RBAC

**4. Blue/Green Deployment Capability**
- ✅ Argo Rollouts controller installed
- ✅ Blue/Green deployment strategy configured
- ✅ Manual promotion control for production safety
- ✅ Instant rollback capability
- ✅ Health checks and analysis templates

### Access Points

- **ArgoCD UI**: LoadBalancer IP (port 443) or localhost:8080
- **Grafana**: localhost:3000 (admin/password from secret)
- **Prometheus**: localhost:9090
- **Applications**: LoadBalancer IPs from respective namespaces

### Key Benefits

1. **Zero-downtime deployments** through GitOps workflow with Blue/Green strategy
2. **Environment isolation** with separate namespaces
3. **Complete observability** with metrics, logs, and dashboards
4. **Automated synchronization** between Git and Kubernetes
5. **Production-ready monitoring** with pre-configured dashboards
6. **Centralized logging** with Loki integration
7. **Instant rollback capability** with Argo Rollouts
8. **Manual promotion control** for production safety

### Process Flow Summary

Each step builds on the previous:

1. **Install ArgoCD** → GitOps workflow foundation
2. **Deploy applications** → Multi-environment setup
3. **Install monitoring** → Complete observability
4. **Install Argo Rollouts** → Blue/Green deployment capability
5. **Configure rollouts** → Zero-downtime deployments
6. **Test and promote** → Production-ready deployments

This complete setup provides a robust foundation for modern Kubernetes application deployment with GitOps, monitoring, and advanced deployment strategies.


## Kubernetes and EKS Glossary

### Core Kubernetes Concepts

**Cluster**
- A set of worker machines (nodes) that run containerized applications
- Managed by a control plane that makes global decisions about the cluster
- In EKS, AWS manages the control plane for you

**Node**
- A worker machine in Kubernetes (can be a VM or physical machine)
- Each node contains services necessary to run pods
- In EKS, these are EC2 instances managed by AWS

**Pod**
- The smallest deployable unit in Kubernetes
- Contains one or more containers that share storage and network
- Pods are ephemeral - they can be created and destroyed as needed

**Container**
- A lightweight, portable unit that packages an application and its dependencies
- Runs inside pods on Kubernetes nodes
- Based on containerization technology like Docker

**Namespace**
- A virtual cluster within a physical cluster
- Provides isolation for resources and allows multiple teams to use the same cluster
- Examples: `production`, `staging`, `monitoring`

### Kubernetes Resources

**Deployment**
- Manages a set of replica pods
- Ensures desired number of pods are running
- Handles rolling updates and rollbacks

**Service**
- An abstraction that defines a logical set of pods and a policy to access them
- Provides stable network endpoint for pods
- Types: ClusterIP, NodePort, LoadBalancer

**ConfigMap**
- Stores non-confidential data in key-value pairs
- Allows you to decouple configuration from container images

**Secret**
- Stores sensitive data like passwords, tokens, and keys
- Similar to ConfigMap but with additional security features

### AWS EKS Specific

**EKS (Elastic Kubernetes Service)**
- AWS managed Kubernetes service
- Handles control plane management, scaling, and updates
- Integrates with AWS services like IAM, VPC, and ELB

**LoadBalancer**
- AWS service that distributes incoming traffic across multiple targets
- In EKS, creates AWS Application Load Balancer (ALB) or Network Load Balancer (NLB)
- Provides external access to services

**VPC (Virtual Private Cloud)**
- Isolated network environment in AWS
- EKS clusters run within your VPC for network isolation and security

### GitOps and CI/CD

**GitOps**
- Operational model that uses Git as the single source of truth
- Declarative approach where desired state is stored in Git
- Automated synchronization between Git and Kubernetes

**ArgoCD**
- Declarative continuous delivery tool for Kubernetes
- Monitors Git repositories and automatically syncs applications
- Provides web UI and CLI for application management

**Argo Rollouts**
- Kubernetes controller and set of CRDs for advanced deployment strategies
- Supports Blue/Green, Canary, and Progressive delivery
- Provides automated rollback capabilities

### Package Management

**Helm**
- Package manager for Kubernetes
- Uses charts (packages) to define, install, and upgrade applications
- Simplifies complex application deployments

**Chart**
- A Helm package containing all resource definitions needed to run an application
- Includes templates, default values, and metadata

**Repository**
- A collection of Helm charts
- Can be public (like Helm Hub) or private
- Examples: `prometheus-community`, `grafana`

### Monitoring and Observability

**Prometheus**
- Open-source monitoring and alerting toolkit
- Collects metrics from configured targets at given intervals
- Stores time-series data and provides querying capabilities

**Grafana**
- Open-source analytics and monitoring platform
- Creates dashboards and visualizations from various data sources
- Integrates with Prometheus, Loki, and other data sources

**Loki**
- Log aggregation system designed for microservices
- Collects, stores, and queries logs from various sources
- Integrates with Grafana for log visualization

**Metrics**
- Quantitative measurements of system behavior
- Examples: CPU usage, memory consumption, request rates
- Collected by Prometheus and displayed in Grafana

**Logs**
- Text records of events that happen in applications
- Collected by Loki and can be searched and visualized
- Essential for debugging and troubleshooting

### Deployment Strategies

**Blue/Green Deployment**
- Deployment strategy that runs two identical production environments
- Blue: Current production version
- Green: New version being tested
- Instant switch between versions with zero downtime

**Rolling Update**
- Default Kubernetes deployment strategy
- Gradually replaces old pods with new ones
- Ensures continuous availability during updates

**Canary Deployment**
- Gradual rollout strategy that routes a small percentage of traffic to new version
- Allows testing new version with real users
- Can be automatically promoted or rolled back based on metrics

### Networking

**Ingress**
- API object that manages external access to services
- Provides HTTP and HTTPS routing to services
- Can provide load balancing, SSL termination, and name-based virtual hosting

**Service Mesh**
- Infrastructure layer for microservices communication
- Handles service-to-service communication, load balancing, and security
- Examples: Istio, Linkerd

### Security

**RBAC (Role-Based Access Control)**
- Authorization mechanism that restricts access based on user roles
- Defines what actions users can perform on which resources
- Essential for multi-tenant Kubernetes environments

**Service Account**
- Identity for processes running in pods
- Provides authentication and authorization for API access
- Can be bound to specific roles and permissions

### Storage

**Persistent Volume (PV)**
- Storage resource in the cluster
- Can be provisioned statically or dynamically
- Examples: EBS volumes, NFS, local storage

**Persistent Volume Claim (PVC)**
- Request for storage by a user
- Binds to a Persistent Volume
- Provides storage abstraction for pods

This glossary covers the essential concepts and tools used throughout the complete EKS GitOps setup guide, providing a foundation for understanding Kubernetes and cloud-native technologies.
