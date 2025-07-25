## Lab Prerequisites and Environment Setup

### Required Tools

- **Kubernetes cluster** (v1.19+ recommended)
- **Helm 3.7+** installed and configured
- **kubectl** configured to access your cluster
- **Git** for repository operations

### Initial Setup Commands

```bash
# Verify Helm installation
helm version

# Verify Kubernetes connectivity
kubectl cluster-info

# Create dedicated namespace for labs
kubectl create namespace monitoring
kubectl create namespace logging
kubectl create namespace argocd
```

## Demo 1: Monitoring with Prometheus and Grafana

### Step 1: Install Prometheus Stack Using Helm

```bash
# Add Prometheus community repository
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### Step 2: Deploy Complete Monitoring Stack

The `kube-prometheus-stack` chart includes Prometheus, Grafana, and Alertmanager:

```bash
# Install the complete monitoring stack
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --set prometheus.prometheusSpec.storageSpec.volumeClaimTemplate.spec.resources.requests.storage=10Gi \
  --set grafana.adminPassword=admin123 \
  --set grafana.service.type=NodePort
```

### Step 3: Verify Installation

```bash
# Check deployed resources
kubectl get all -n monitoring

# Get Grafana admin password (if not set)
kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode

# Port-forward to access Grafana
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80
```

### Lab Exercise 1: Custom ServiceMonitor

Create a custom application with metrics:

```yaml
# demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: demo-app
  template:
    metadata:
      labels:
        app: demo-app
    spec:
      containers:
      - name: demo-app
        image: prom/node-exporter:latest
        ports:
        - containerPort: 9100
---
apiVersion: v1
kind: Service
metadata:
  name: demo-app-service
  namespace: monitoring
  labels:
    app: demo-app
spec:
  ports:
  - port: 9100
    targetPort: 9100
    name: metrics
  selector:
    app: demo-app
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: demo-app-monitor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: demo-app
  endpoints:
  - port: metrics
```

### Lab Tasks

1. Deploy the demo application
2. Verify metrics are being scraped by Prometheus
3. Create a Grafana dashboard for the custom metrics
4. Set up an alert rule for high CPU usage

## Demo 2: Logging with EFK Stack

### Step 1: Deploy Elasticsearch

```bash
# Add Elastic repository
helm repo add elastic https://helm.elastic.co
helm repo update

# Install Elasticsearch
helm install elasticsearch elastic/elasticsearch \
  --namespace logging \
  --set replicas=1 \
  --set minimumMasterNodes=1 \
  --set resources.requests.memory=1Gi \
  --set resources.limits.memory=2Gi
```

### Step 2: Deploy Kibana

```bash
# Install Kibana
helm install kibana elastic/kibana \
  --namespace logging \
  --set service.type=NodePort \
  --set resources.requests.memory=512Mi
```

### Step 3: Deploy Fluent Bit for Log Collection

```bash
# Add Fluent repository
helm repo add fluent https://fluent.github.io/helm-charts

# Install Fluent Bit
helm install fluent-bit fluent/fluent-bit \
  --namespace logging \
  --set config.outputs[^0].name=es \
  --set config.outputs[^0].match="*" \
  --set config.outputs[^0].host=elasticsearch-master \
  --set config.outputs[^0].port=9200 \
  --set config.outputs[^0].index=kubernetes-logs
```

### Lab Exercise 2: Custom Log Parser

Create a custom Fluent Bit configuration:

```yaml
# fluent-bit-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         1
        Log_Level     info
        Daemon        off
        Parsers_File  parsers.conf

    [INPUT]
        Name              tail
        Path              /var/log/containers/*.log
        Parser            docker
        Tag               kube.*
        Refresh_Interval  5
        Mem_Buf_Limit     5MB

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token

    [OUTPUT]
        Name  es
        Match *
        Host  elasticsearch-master
        Port  9200
        Index kubernetes-logs
        Type  _doc
```

### Lab Tasks

1. Deploy a sample application that generates logs
2. Configure log parsing for application-specific formats
3. Create Kibana visualizations and dashboards
4. Set up log-based alerts

## Demo 3: GitOps with ArgoCD

### Step 1: Install ArgoCD Using Helm

```bash
# Add ArgoCD repository
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD
helm install argocd argo/argo-cd \
  --namespace argocd \
  --set server.service.type=NodePort \
  --set server.extraArgs[0]="--insecure"
```

### Step 2: Access ArgoCD UI

```bash
# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Port-forward to access UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

### Step 3: Create Sample GitOps Repository

Create a repository structure:

```
├── applications/
│   ├── dev/
│   │   └── my-app.yaml
│   ├── staging/
│   │   └── my-app.yaml
│   └── prod/
│       └── my-app.yaml
└── helm-charts/
    └── my-app/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
```

### Lab Exercise 3: ArgoCD Application with Helm

Create an ArgoCD Application manifest:

```yaml
# argocd-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/demo-app'
    path: helm-charts/demo-app
    targetRevision: main
    helm:
      values: |
        replicaCount: 2
        image:
          repository: nginx
          tag: "1.21"
        service:
          type: ClusterIP
          port: 80
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

### Step 4: Deploy App of Apps Pattern

```yaml
# app-of-apps.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: app-of-apps
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/your-org/argocd-apps'
    path: applications
    targetRevision: main
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Lab Tasks

1. Create a Git repository with Helm charts
2. Configure ArgoCD to watch the repository
3. Implement environment-specific configurations
4. Test automated sync and rollback capabilities
5. Set up notifications for deployment events

## Practical Lab Scenarios

### Scenario 1: End-to-End Monitoring Pipeline

1. Deploy a multi-tier application (frontend, backend, database)
2. Configure Prometheus to scrape metrics from all tiers[^2]
3. Set up Grafana dashboards for each service
4. Create alert rules for critical metrics
5. Configure Alertmanager for notifications

### Scenario 2: Centralized Logging Solution

1. Deploy applications across multiple namespaces
2. Configure Fluent Bit to collect logs from all applications[^3][^4]
3. Parse different log formats (JSON, plain text, multiline)
4. Create Kibana index patterns and visualizations
5. Set up log-based monitoring and alerting

### Scenario 3: GitOps Workflow Implementation

1. Create separate Git repositories for application code and configurations
2. Set up CI pipeline to build and push container images[^5][^6]
3. Configure ArgoCD to deploy from configuration repository
4. Implement promotion workflow across environments
5. Test disaster recovery and rollback procedures

## Validation Commands

### Monitoring Validation

```bash
# Check Prometheus targets
kubectl port-forward -n monitoring svc/prometheus-kube-prometheus-prometheus 9090:9090

# Verify Grafana access
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80

# Check alert rules
kubectl get prometheusrules -n monitoring
```

### Logging Validation

```bash
# Verify Elasticsearch cluster health
kubectl port-forward -n logging svc/elasticsearch-master 9200:9200
curl http://localhost:9200/_cluster/health

# Check Fluent Bit logs
kubectl logs -n logging -l app.kubernetes.io/name=fluent-bit

# Access Kibana
kubectl port-forward -n logging svc/kibana-kibana 5601:5601
```

### GitOps Validation

```bash
# Check ArgoCD applications
kubectl get applications -n argocd

# Verify sync status
argocd app list

# Check application health
argocd app get demo-app
```
