# Advanced Kubernetes Deployment and Scaling - Labs and Demos

## **Demo 2: Canary Deployment with Native Kubernetes**

### **Objective**

Implement canary deployment using Kubernetes native features without service mesh.

### **Step 1: Deploy Production Version (v1)**

production-v1.yaml

```yaml
# production-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v1
  labels:
    app: webapp
    version: v1
spec:
  replicas: 9  # 90% of traffic
  selector:
    matchLabels:
      app: webapp
      version: v1
  template:
    metadata:
      labels:
        app: webapp
        version: v1
    spec:
      containers:
      - name: webapp
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v1.0"
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "<h1>Version 1.0</h1>" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp  # Selects both v1 and v2 pods
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer
```

### **Step 2: Deploy Canary Version (v2)**

canary-v2.yaml

```yaml
# canary-v2.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-v2
  labels:
    app: webapp
    version: v2
spec:
  replicas: 1  # 10% of traffic initially
  selector:
    matchLabels:
      app: webapp
      version: v2
  template:
    metadata:
      labels:
        app: webapp
        version: v2
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "v2.0"
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "<h1>Version 2.0</h1>" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
```

### **Step 3: Monitor Canary Deployment**

Deploy both versions

```bash
kubectl apply -f production-v1.yaml
kubectl apply -f canary-v2.yaml
```

Check pod distribution

```bash
kubectl get pods -l app=webapp --show-labels
```

Test traffic distribution

create test-client.sh and run it inside busybox pod.

```bash
#!/bin/sh
i=1
while [ $i -le 100 ]; do
  wget -qO- http://webapp-service | head -1
  sleep 1
done
```

### **Step 4: Gradually Increase Canary Traffic**

Phase 1: 10% canary (1 v2 pod, 9 v1 pods)

```bash
kubectl scale deployment webapp-v2 --replicas=2
kubectl scale deployment webapp-v1 --replicas=8
```

Phase 2: 25% canary (2 v2 pods, 6 v1 pods)

```bash
kubectl scale deployment webapp-v2 --replicas=3
kubectl scale deployment webapp-v1 --replicas=6
```

Phase 3: 50% canary (3 v2 pods, 3 v1 pods)

```bash
kubectl scale deployment webapp-v2 --replicas=5
kubectl scale deployment webapp-v1 --replicas=0
```

Complete rollout: 100% canary

```bash
kubectl delete deployment webapp-v1
```

### **Step 5: Automated Canary with Script**

canary-rollout.sh

```bash
#!/bin/bash

CANARY_DEPLOYMENT="webapp-v2"
STABLE_DEPLOYMENT="webapp-v1"
TOTAL_REPLICAS=10

# Function to update replica counts

update_replicas() {
local canary_percent=$1
local canary_replicas=$(( TOTAL_REPLICAS * canary_percent / 100 ))
local stable_replicas=$(( TOTAL_REPLICAS - canary_replicas ))

echo "Setting canary to $canary_replicas replicas, stable to $stable_replicas replicas"
kubectl scale deployment $CANARY_DEPLOYMENT --replicas=$canary_replicas
kubectl scale deployment $STABLE_DEPLOYMENT --replicas=$stable_replicas

#Wait for rollout

kubectl rollout status deployment $CANARY_DEPLOYMENT

sleep 30 # Monitor for 30 seconds
}


#Gradual rollout

update_replicas 10
update_replicas 25
update_replicas 50
update_replicas 75
update_replicas 100
```

Cleanup stable deployment

```bash
kubectl delete deployment $STABLE_DEPLOYMENT
```
