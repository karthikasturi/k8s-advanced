# Day 3: Kubernetes Advanced

## Demo 1: Helm Charts and Package Management

## Prerequisites Setup

```bash
# Install Helm (if not already installed)
curl https://get.helm.sh/helm-v3.12.0-linux-amd64.tar.gz -o helm.tar.gz
tar -zxvf helm.tar.gz
sudo mv linux-amd64/helm /usr/local/bin/helm
helm version

# Add Helm repositories
helm repo add stable https://charts.helm.sh/stable
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

### 1.1: Working with Existing Charts

```bash
# Search for charts
helm search repo nginx
helm search hub wordpress

# Show chart information
helm show chart bitnami/nginx
helm show values bitnami/nginx

# Install a chart
helm install my-nginx bitnami/nginx
kubectl get pods
kubectl get services

# List releases
helm list

# Upgrade release with custom values
helm upgrade my-nginx bitnami/nginx --set service.type=NodePort
helm history my-nginx

# Rollback release
helm rollback my-nginx 1
```

### 1.2: Creating Custom Helm Chart

```bash
# Create new chart
helm create webapp
cd webapp

# Show chart structure
tree .
cat Chart.yaml
cat values.yaml
```

**Edit values.yaml:**

```yaml
replicaCount: 2
image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.21"

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: ""
  annotations:
    kubernetes.io/ingress.class: nginx
  hosts:
    - host: webapp.local
      paths:
        - path: /
          pathType: Prefix
```

**Customize templates/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "webapp.fullname" . }}
  labels:
    {{- include "webapp.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "webapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "webapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          ports:
            - name: http
              containerPort: 80
              protocol: TCP
          env:
            - name: APP_VERSION
              value: {{ .Chart.Version | quote }}
```

### 1.3: Template Testing and Deployment

```bash
# Validate chart
helm lint webapp

# Test template rendering
helm template webapp ./webapp
helm template webapp ./webapp --values custom-values.yaml

# Dry run installation
helm install webapp ./webapp --dry-run --debug

# Install custom chart
helm install webapp ./webapp
kubectl get all -l app.kubernetes.io/name=webapp

# Package chart
helm package webapp
ls *.tgz 
```
