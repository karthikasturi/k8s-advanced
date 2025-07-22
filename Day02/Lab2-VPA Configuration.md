## **Lab Exercise 2: VPA Configuration**

### **Objective**

Configure Vertical Pod Autoscaler to optimize resource allocation for applications.

### **Step 1: Install VPA**

Clone VPA repository

```bash
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
```

Install VPA components

```bash
./hack/vpa-up.sh
```

Verify VPA installation

```bash
kubectl get pods -n kube-system | grep vpa
```

### **Step 2: Deploy Application for VPA Testing**

vpa-demo-app.yaml

```yaml
# vpa-demo-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo
  labels:
    app: vpa-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo
  template:
    metadata:
      labels:
        app: vpa-demo
    spec:
      containers:
      - name: container
        image: ubuntu:22.04
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            # Simulate CPU and memory usage
            timeout 10s yes >/dev/null
            sleep 20s
          done
        resources:
          requests:
            cpu: 100m
            memory: 50Mi
          limits:
            cpu: 200m
            memory: 100Mi
```

### **Step 3: Create VPA in Recommendation Mode**

vpa-recommender.yaml

```yaml
# vpa-recommender.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo-recommender
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Off"  # Only provide recommendations
  resourcePolicy:
    containerPolicies:
    - containerName: container
      maxAllowed:
        cpu: 1
        memory: 500Mi
      minAllowed:
        cpu: 100m
        memory: 50Mi
      controlledResources: ["cpu", "memory"]
```

### **Step 4: Monitor VPA Recommendations**

Deploy application and VPA

```bash
kubectl apply -f vpa-demo-app.yaml
kubectl apply -f vpa-recommender.yaml
```

Wait for some time to collect metrics

```bash
sleep 300
```

View VPA recommendations

```bash
kubectl describe vpa vpa-demo-recommender
```

Get detailed VPA status

```bash
kubectl get vpa vpa-demo-recommender -o yaml
```

### **Step 5: Enable Automatic VPA Updates**

vpa-auto-update.yaml

```yaml
# vpa-auto-update.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo-auto
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo
  updatePolicy:
    updateMode: "Auto"  # Automatically update pods
  resourcePolicy:
    containerPolicies:
    - containerName: container
      maxAllowed:
        cpu: 1
        memory: 500Mi
      minAllowed:
        cpu: 100m
        memory: 50Mi
      controlledResources: ["cpu", "memory"]
      controlledValues: RequestsAndLimits
```

### **Step 6: Monitor VPA in Action**

Update VPA to auto mode

```bash
kubectl apply -f vpa-auto-update.yaml
```

Watch pods get updated with new resource requirements

```bash
kubectl get pods -l app=vpa-demo -w
```

Check updated resource requests

```bash
kubectl describe pods -l app=vpa-demo | grep -A 10 "Requests:"
```

### **Step 7: VPA with Initial Mode**

vpa-initial-mode.yaml

```yaml
# vpa-initial-mode.yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: vpa-demo-initial
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: vpa-demo-initial
  updatePolicy:
    updateMode: "Initial"  # Set resources only for new pods
  resourcePolicy:
    containerPolicies:
    - containerName: container
      maxAllowed:
        cpu: 1
        memory: 500Mi
      minAllowed:
        cpu: 200m
        memory: 100Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vpa-demo-initial
spec:
  replicas: 2
  selector:
    matchLabels:
      app: vpa-demo-initial
  template:
    metadata:
      labels:
        app: vpa-demo-initial
    spec:
      containers:
      - name: container
        image: ubuntu:22.04
        command: ["/bin/sh"]
        args: ["-c", "while true; do sleep 30; done"]
        # Note: No resource requests defined initially
```
