# Advanced Kubernetes Deployment and Scaling - Labs and Demos

## **Demo 3: Rolling Updates and Rollbacks**

### **Objective**

Demonstrate rolling update process and rollback mechanisms in Kubernetes.

### **Step 1: Create Initial Deployment**

rolling-update-demo.yaml

```yaml
# rolling-update-demo.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
  annotations:
    deployment.kubernetes.io/revision: "1"
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2     # 20% unavailable
      maxSurge: 2           # 20% extra pods
  selector:
    matchLabels:
      app: rolling-demo
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
      - name: app
        image: nginx:1.19
        ports:
        - containerPort: 80
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
```

### **Step 2: Deploy and Monitor Initial Version**

Deploy initial version

```bash
kubectl apply -f rolling-update-demo.yaml
```

Check deployment status

```bash
kubectl rollout status deployment/rolling-demo
```

View deployment history

```bash
kubectl rollout history deployment/rolling-demo
```

### **Step 3: Trigger Rolling Update**

Update to new image version

```bash
kubectl set image deployment/rolling-demo app=nginx:1.20
```

Watch rolling update in real-time

```bash
kubectl rollout status deployment/rolling-demo --watch
```

Monitor pods during update

```bash
kubectl get pods -l app=rolling-demo -w
```

### **Step 4: Monitor Rolling Update Process**

Check ReplicaSets

```bash
kubectl get rs -l app=rolling-demo
```

View detailed deployment events

```bash
kubectl describe deployment rolling-demo
```

Check rollout history

```bash
kubectl rollout history deployment/rolling-demo
```

### **Step 5: Demonstrate Rollback**

Rollback to previous version

```bash
kubectl rollout undo deployment/rolling-demo
```

Check rollback status

```bash
kubectl rollout status deployment/rolling-demo
```

Rollback to specific revision

```bash
kubectl rollout undo deployment/rolling-demo --to-revision=1
```

View updated history

```bash
kubectl rollout history deployment/rolling-demo
```

### **Step 6: Advanced Rolling Update Configuration**

advanced-rolling-update.yaml

```yaml
# advanced-rolling-update.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: advanced-rolling-demo
spec:
  replicas: 15
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1     # Only 1 pod unavailable at a time
      maxSurge: 3           # Up to 3 extra pods during update
  minReadySeconds: 10       # Wait 10 seconds after pod is ready
  progressDeadlineSeconds: 300  # Fail deployment after 5 minutes
  revisionHistoryLimit: 5   # Keep 5 old ReplicaSets
  selector:
    matchLabels:
      app: advanced-rolling-demo
  template:
    metadata:
      labels:
        app: advanced-rolling-demo
    spec:
      containers:
      - name: app
        image: nginx:1.19
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 2
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
```

```bash
kubectl apply -f advanced-rolling-update.yaml
```

```bash
kubectl rollout status deployment/advanced-rolling-demo
```

```bash
kubectl rollout status deployment/advanced-rolling-demo
```

```bash
kubectl get rs -l app=advanced-rolling-demo --watch
```

```bash
kubectl get pods -l app=advanced-rolling-demo -w
```
