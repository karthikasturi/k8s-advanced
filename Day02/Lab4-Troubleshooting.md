## **Lab Exercise 4: Troubleshooting Autoscaling Issues**

### **Objective**

Practice diagnosing and fixing common autoscaling problems.

### **Common Issue 1: Missing Resource Requests**

broken-hpa-no-requests.yaml

```yaml
# broken-hpa-no-requests.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: broken-hpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: broken-hpa-demo
  template:
    metadata:
      labels:
        app: broken-hpa-demo
    spec:
      containers:
      - name: app
        image: nginx:1.20
        # Missing resource requests!
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: broken-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: broken-hpa-demo
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

### **Debugging Steps:**

Deploy broken configuration

```bash
kubectl apply -f broken-hpa-no-requests.yaml
```

Check HPA status (should show unknown/0%)

```bash
kubectl get hpa broken-hpa
```

Describe HPA to see error messages

```bash
kubectl describe hpa broken-hpa
```

Fix by adding resource requests

```bash
kubectl patch deployment broken-hpa-demo -p '{"spec":{"template":{"spec":{"containers":[{"name":"app","resources":{"requests":{"cpu":"100m","memory":"128Mi"}}}]}}}}'
```

Verify fix

```bash
kubectl get hpa broken-hpa
```

### **Common Issue 2: Metrics Server Problems**

Check if metrics server is running

```bash
kubectl get pods -n kube-system | grep metrics-server
```

Check metrics server logs

```bash
kubectl logs -n kube-system deployment/metrics-server
```

Test if metrics are available

```bash
kubectl top nodes
kubectl top pods
```

If metrics are not working, restart metrics server

```bash
kubectl rollout restart deployment/metrics-server -n kube-system
```

### **Common Issue 3: HPA Not Scaling**

Check HPA events

```bash
kubectl describe hpa <hpa-name>
```

Check deployment events

```bash
kubectl describe deployment <deployment-name>
```

Check pod resource usage

```bash
kubectl top pods -l app=<app-label>
```

Check HPA conditions

```bash
kubectl get hpa <hpa-name> -o yaml | grep -A 10 conditions
```
