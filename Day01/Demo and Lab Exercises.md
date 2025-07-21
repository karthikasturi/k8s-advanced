# 

# Demo and Lab Exercises: Kubernetes Scheduling and Taints/Tolerations

Here's a comprehensive collection of demonstrations and hands-on lab exercises to deepen your understanding of Kubernetes scheduling, taints, and tolerations.

## **Demo 1: Understanding the Default Scheduler**

### Live Demonstration Script

**Objective:** Show how the Kubernetes scheduler works by default

```bash
# 1. Create a deployment without any scheduling constraints
kubectl create deployment demo-nginx --image=nginx --replicas=3

# 2. Watch the scheduling process in real-time
kubectl get pods -o wide --watch

# 3. Show node resource usage
kubectl top nodes

# 4. Examine scheduler decision-making
kubectl describe pod <pod-name> | grep -A 10 Events
```

**Key Points to Highlight:**

- The scheduler considers resource availability
- Pods are distributed across available nodes
- The scheduling process happens in two phases: filtering and scoring

## **Lab Exercise 1: Node Affinity Basics**

### **Scenario:** Deploy applications to specific hardware types

**Step 1: Label your nodes**

```bash
# Label nodes based on hardware characteristics
kubectl label nodes node-1 hardware=gpu
kubectl label nodes node-2 hardware=ssd
kubectl label nodes node-3 hardware=standard
```

**Step 2: Create a pod with node affinity**

```yaml
# gpu-workload.yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-workload
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: hardware
            operator: In
            values:
            - gpu
  containers:
  - name: gpu-app
    image: tensorflow/tensorflow:latest-gpu
```

**Step 3: Deploy and verify**

```bash
kubectl apply -f gpu-workload.yaml
kubectl get pod gpu-workload -o wide
```

**Expected Result:** Pod should only schedule on the GPU-labeled node.

## **Lab Exercise 2: Advanced Taints and Tolerations**

### **Scenario:** Create a dedicated node pool for production workloads

**Step 1: Taint production nodes**

```bash
# Taint nodes for production use
kubectl taint nodes prod-node-1 environment=production:NoSchedule
kubectl taint nodes prod-node-2 environment=production:NoSchedule

# Add additional taint for critical workloads only
kubectl taint nodes prod-node-1 workload=critical:NoExecute
```

**Step 2: Create production deployment with tolerations**

```yaml
# production-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: production-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
    spec:
      tolerations:
      - key: environment
        operator: Equal
        value: production
        effect: NoSchedule
      containers:
      - name: app
        image: nginx:1.20
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

**Step 3: Create critical workload with multiple tolerations**

```yaml
# critical-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: critical-app
  template:
    metadata:
      labels:
        app: critical-app
    spec:
      tolerations:
      - key: environment
        operator: Equal
        value: production
        effect: NoSchedule
      - key: workload
        operator: Equal
        value: critical
        effect: NoExecute
      containers:
      - name: critical-service
        image: redis:6.2
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
```

**Verification Commands:**

```bash
# Deploy both applications
kubectl apply -f production-app.yaml
kubectl apply -f critical-app.yaml

# Check where pods are scheduled
kubectl get pods -o wide

# Try deploying a regular pod (should avoid tainted nodes)
kubectl run test-pod --image=busybox --command -- sleep 3600
kubectl get pod test-pod -o wide
```

## **Lab Exercise 3: Pod Anti-Affinity for High Availability**

### **Scenario:** Ensure web application pods are spread across different nodes

```yaml
# ha-web-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ha-web-app
spec:
  replicas: 4
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        version: v1
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - web
            topologyKey: "kubernetes.io/hostname"
        podAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - database
              topologyKey: "topology.kubernetes.io/zone"
      containers:
      - name: web
        image: nginx:1.20
        ports:
        - containerPort: 80
```

**Testing Commands:**

```bash
kubectl apply -f ha-web-app.yaml
kubectl get pods -l app=web -o wide

# Simulate node failure
kubectl drain node-2 --ignore-daemonsets --delete-emptydir-data
kubectl get pods -l app=web -o wide
```

## **Lab Exercise 4: Taint Effects Deep Dive**

### **Understanding Different Taint Effects**

**Step 1: Set up nodes with different taint effects**

```bash
# NoSchedule effect
kubectl taint nodes node-1 test=noschedule:NoSchedule

# PreferNoSchedule effect  
kubectl taint nodes node-2 test=prefernoschedule:PreferNoSchedule

# NoExecute effect (will evict existing pods)
kubectl taint nodes node-3 test=noexecute:NoExecute
```

**Step 2: Test each effect**

```yaml
# Create test pods to observe behavior
apiVersion: v1
kind: Pod
metadata:
  name: test-noschedule
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
  tolerations:
  - key: test
    value: noschedule
    effect: NoSchedule
    operator: Equal
---
apiVersion: v1
kind: Pod
metadata:
  name: test-noexecute
spec:
  containers:
  - name: test
    image: busybox
    command: ['sleep', '3600']
  tolerations:
  - key: test
    value: noexecute
    effect: NoExecute
    operator: Equal
    tolerationSeconds: 300  # Tolerate for 5 minutes before eviction
```

## **Lab Exercise 5: Priority Classes and Preemption**

### **Scenario:** Ensure critical workloads get resources when needed

**Step 1: Create priority classes**

```yaml
# priority-classes.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "High priority class for critical services"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
globalDefault: false
description: "Low priority class for batch jobs"
```

**Step 2: Deploy low priority workload**

```yaml
# low-priority-job.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-job
spec:
  replicas: 10
  selector:
    matchLabels:
      app: batch
  template:
    metadata:
      labels:
        app: batch
    spec:
      priorityClassName: low-priority
      containers:
      - name: worker
        image: busybox
        command: ['sleep', '3600']
        resources:
          requests:
            memory: "500Mi"
            cpu: "500m"
```

**Step 3: Deploy high priority workload to trigger preemption**

```yaml
# high-priority-service.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: critical-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: critical
  template:
    metadata:
      labels:
        app: critical
    spec:
      priorityClassName: high-priority
      containers:
      - name: service
        image: nginx:1.20
        resources:
          requests:
            memory: "1Gi"
            cpu: "1"
```

## **Troubleshooting Lab Exercise**

### **Common Scheduling Issues and Solutions**

**Exercise: Debug scheduling failures**

1. **Create a pod that cannot be scheduled**
   
   ```yaml
   apiVersion: v1
   kind: Pod
   metadata:
   name: impossible-pod
   spec:
   containers:
   - name: app
    image: nginx
    resources:
      requests:
        memory: "100Gi"  # Impossible memory requirement
        cpu: "50"        # Impossible CPU requirement
   ```

2. **Debug commands:**
   
   ```bash
   # Check pod status
   kubectl get pods
   kubectl describe pod impossible-pod
   ```

# Check scheduler logs

```bash
kubectl logs -n kube-system -l component=kube-scheduler
```

# Check node resources

```bash
kubectl describe nodes
```

## **Cleanup Commands**

```bash
# Remove all test resources
kubectl delete deployments --all
kubectl delete pods --all
kubectl delete priorityclasses high-priority low-priority

# Remove taints
kubectl taint nodes --all test:NoSchedule-
kubectl taint nodes --all test:PreferNoSchedule-
kubectl taint nodes --all test:NoExecute-
kubectl taint nodes --all environment:NoSchedule-
kubectl taint nodes --all workload:NoExecute-

# Remove labels
kubectl label nodes --all hardware-
```
