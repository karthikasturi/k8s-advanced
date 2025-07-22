## **Lab Exercise 3: Combined HPA and VPA Setup**

### **Objective**

Demonstrate how HPA and VPA can work together (with careful configuration).

### **Step 1: Deploy Application with HPA on CPU, VPA on Memory**

combined-autoscaling.yaml

```yaml
# combined-autoscaling.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: combined-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: combined-demo
  template:
    metadata:
      labels:
        app: combined-demo
    spec:
      containers:
      - name: app
        image: nginx:1.20
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
---
# HPA focusing on CPU only
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: combined-demo-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: combined-demo
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
---
# VPA focusing on memory only
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: combined-demo-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: combined-demo
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: app
      controlledResources: ["memory"]  # Only control memory
      maxAllowed:
        memory: 1Gi
      minAllowed:
        memory: 64Mi
```

### **Step 2: Test Combined Scaling**

Deploy combined setup

```bash
kubectl apply -f combined-autoscaling.yaml
```

Generate CPU load (triggers HPA)

```bash
kubectl run cpu-loader --rm -i --tty --image=busybox --restart=Never -- /bin/sh
```

Inside pod:

```bash
while true; do wget -q -O- http://combined-demo-service; done
```

Generate memory pressure (triggers VPA)

```bash
kubectl exec -it deployment/combined-demo -- /bin/sh
```

Inside pod:

```bash
stress --vm 1 --vm-bytes 256M --timeout 60s
```

Monitor both autoscalers

```bash
kubectl get hpa combined-demo-hpa --watch
kubectl get vpa combined-demo-vpa --watch
```
