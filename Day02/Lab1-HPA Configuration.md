## **Lab Exercise 1: HPA Configuration**

### **Objective**

Configure Horizontal Pod Autoscaler for a web application and test autoscaling behavior.

### **Prerequisites**

Ensure metrics server is installed

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

For local clusters, you might need to add --kubelet-insecure-tls

```bash
kubectl patch deployment metrics-server -n kube-system --type='json' -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--kubelet-insecure-tls"}]'
```

Verify metrics server

```bash
kubectl get deployment metrics-server -n kube-system
kubectl top nodes
```

### **Step 1: Deploy Sample Application**

webapp-for-hpa.yaml

```yaml
# webapp-for-hpa.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-hpa-demo
  template:
    metadata:
      labels:
        app: webapp-hpa-demo
    spec:
      containers:
      - name: webapp
        image: k8s.gcr.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-hpa-service
spec:
  selector:
    app: webapp-hpa-demo
  ports:
  - port: 80
    targetPort: 80
```

### **Step 2: Create Basic HPA**

basic-hpa.yaml

```yaml
# basic-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp-hpa-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### **Step 3: Deploy Application and HPA**

Deploy application

```bash
kubectl apply -f webapp-for-hpa.yaml
```

Verify deployment

```bash
kubectl get pods -l app=webapp-hpa-demo -w
```

Deploy HPA

```bash
kubectl apply -f basic-hpa.yaml
```

Check HPA status

```bash
kubectl get hpa webapp-hpa
```

### **Step 4: Generate Load and Monitor Scaling**

Generate load using busybox

```bash
kubectl run load-generator --rm -i --tty --image=busybox --restart=Never -- /bin/sh
```

Inside the load generator pod, run:

```bash
while true; do wget -q -O- http://webapp-hpa-service; done
```

In another terminal, monitor HPA and pods

```bash
kubectl get hpa webapp-hpa --watch
kubectl get pods -l app=webapp-hpa-demo --watch
```

### **Step 5: Advanced HPA with Multiple Metrics**

advanced-hpa.yaml

```yaml
# advanced-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-advanced-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp-hpa-demo
  minReplicas: 3
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 60
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
      - type: Pods
        value: 4
        periodSeconds: 15
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
      - type: Pods
        value: 2
        periodSeconds: 60
      selectPolicy: Min
```

### **Step 6: Custom Metrics HPA (Optional)**

custom-metrics-hpa.yaml

```yaml
# custom-metrics-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: webapp-custom-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: webapp-hpa-demo
  minReplicas: 2
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
  - type: External
    external:
      metric:
        name: pubsub.googleapis.com|subscription|num_undelivered_messages
        selector:
          matchLabels:
            resource.labels.subscription_id: "my-subscription"
      target:
        type: AverageValue
        averageValue: "30"
```
