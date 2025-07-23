### LAB 1: Deploying Applications with Helm

**Objective**: Create and deploy a complete web application stack using Helm charts.

## Lab Setup

```bash
# Create lab directory
mkdir -p ~/k8s-labs/day3/helm-lab
cd ~/k8s-labs/day3/helm-lab

# Verify Helm installation
helm version
```

## Task 1: Deploy WordPress using Helm

```bash
# Add bitnami repository
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Create namespace
kubectl create namespace wordpress

# Check WordPress chart values
helm show values bitnami/wordpress > wordpress-values.yaml
```

**Edit wordpress-values.yaml:**

```yaml
wordpressUsername: admin
wordpressPassword: admin123
wordpressEmail: admin@example.com
wordpressBlogName: "My K8s Blog"

service:
  type: NodePort
  nodePorts:
    http: 30080

persistence:
  enabled: true
  size: 8Gi

mariadb:
  auth:
    rootPassword: "root123"
    database: "wordpress"
    username: "wordpress"
    password: "wordpress123"
  primary:
    persistence:
      enabled: true
      size: 8Gi
```

```bash
# Install WordPress with custom values
helm install my-wordpress bitnami/wordpress \
  --namespace wordpress \
  --values wordpress-values.yaml

# Monitor deployment
kubectl get pods -n wordpress -w

# Get WordPress URL
export NODE_PORT=$(kubectl get svc my-wordpress -n wordpress -o jsonpath='{.spec.ports[0].nodePort}')
export NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
echo "WordPress URL: http://$NODE_IP:$NODE_PORT"

kubectl get svc -n wordpress
kubectl port-forward svc/my-wordpress 8080:80 -n wordpress

# Check release status
helm status my-wordpress -n wordpress
helm list -n wordpress
```

## Task 2: Create Custom Application Chart

```bash
# Create new chart for a simple web app
helm create myapp
cd myapp

# Remove default nginx configurations
rm -rf templates/tests/ templates/serviceaccount.yaml
```

**Edit Chart.yaml:**

```yaml
apiVersion: v2
name: myapp
description: A Helm chart for custom web application
type: application
version: 0.1.0
appVersion: "1.0.0"
maintainers:
- name: "Your Name"
  email: "your.email@example.com"
keywords:
- web
- application
- demo
```

**Edit values.yaml:**

```yaml
replicaCount: 2

image:
  repository: httpd
  pullPolicy: IfNotPresent
  tag: "2.4"

nameOverride: ""
fullnameOverride: ""

service:
  type: ClusterIP
  port: 80
  targetPort: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
  hosts:
    - host: myapp.local
      paths:
        - path: /
          pathType: Prefix
  tls: []

resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

nodeSelector: {}
tolerations: []
affinity: {}

# Custom configurations
config:
  message: "Hello from Kubernetes with Helm!"
  debug: true

healthCheck:
  enabled: true
  path: /health
  initialDelaySeconds: 30
  periodSeconds: 10
```

**Create templates/configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>{{ .Values.config.message }}</title>
        <style>
            body { font-family: Arial, sans-serif; margin: 50px; }
            .container { max-width: 800px; margin: 0 auto; }
            .info { background: #f0f0f0; padding: 20px; border-radius: 5px; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>{{ .Values.config.message }}</h1>
            <div class="info">
                <p><strong>App Version:</strong> {{ .Chart.AppVersion }}</p>
                <p><strong>Chart Version:</strong> {{ .Chart.Version }}</p>
                <p><strong>Release Name:</strong> {{ .Release.Name }}</p>
                <p><strong>Namespace:</strong> {{ .Release.Namespace }}</p>
                <p><strong>Revision:</strong> {{ .Release.Revision }}</p>
                {{- if .Values.config.debug }}
                <p><strong>Debug Mode:</strong> Enabled</p>
                <p><strong>Timestamp:</strong> {{ now | date "2006-01-02 15:04:05" }}</p>
                {{- end }}
            </div>
        </div>
    </body>
    </html>
  health.html: |
    OK
```

**Update templates/deployment.yaml:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- if .Values.healthCheck.enabled }}
          livenessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: http
            initialDelaySeconds: {{ .Values.healthCheck.initialDelaySeconds }}
            periodSeconds: {{ .Values.healthCheck.periodSeconds }}
          readinessProbe:
            httpGet:
              path: {{ .Values.healthCheck.path }}
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
          {{- end }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
          - name: config
            mountPath: /usr/local/apache2/htdocs
      volumes:
      - name: config
        configMap:
          name: {{ include "myapp.fullname" . }}-config
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
```

```bash
# Validate chart
helm lint .

# Test template rendering
helm template myapp . --debug

# Install the chart
helm install myapp . --namespace default

# Verify deployment
kubectl get all -l app.kubernetes.io/name=myapp

# Test the application
kubectl port-forward svc/myapp 8080:80 &
curl http://localhost:8080
curl http://localhost:8080/health

# Package the chart
cd ..
helm package myapp
ls -la *.tgz    
```
