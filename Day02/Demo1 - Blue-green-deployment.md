# Advanced Kubernetes Deployment and Scaling - Labs and Demos

## **Demo 1: Blue/Green Deployment Demonstration**

### **Objective**

Demonstrate how to implement and execute a blue/green deployment strategy in Kubernetes.

### **Prerequisites**

- Kubernetes cluster with at least 2 nodes
- kubectl configured and working
- Basic understanding of Deployments and Services

### **Step 1: Create Blue Environment (Current Version)**

```yaml
# blue-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-blue
  labels:
    app: webapp
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: blue
  template:
    metadata:
      labels:
        app: webapp
        version: blue
    spec:
      containers:
      - name: webapp
        image: nginx:1.20
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "blue-v1.0"
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "<h1>Blue Deployment - Version 1.0</h1><p>This is the BLUE environment</p>" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
spec:
  selector:
    app: webapp
    version: blue  # Initially pointing to blue
  ports:
  - port: 80
    targetPort: 80
  type: LoadBalancer

```

:wq### **Step 2: Deploy Blue Environment**

Deploy blue environment

```bash
kubectl apply -f blue-deployment.yaml
```

Verify blue deployment

```bash
kubectl get pods -l version=blue
kubectl get service webapp-service
```

Test blue environment

```bash
kubectl run -it --rm test-client --image=busybox --restart=Never -- sh
```

run the following command inside the busybox container

```bash
wget -qO- http://webapp-service
```

### **Step 3: Create Green Environment (New Version)**

```yaml
# green-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-green
  labels:
    app: webapp
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webapp
      version: green
  template:
    metadata:
      labels:
        app: webapp
        version: green
    spec:
      containers:
      - name: webapp
        image: nginx:1.21
        ports:
        - containerPort: 80
        env:
        - name: VERSION
          value: "green-v2.0"
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "<h1>Green Deployment - Version 2.0</h1><p>This is the GREEN environment</p>" > /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi

```

### **Step 4: Deploy and Test Green Environment**

```bash
kubectl apply -f green-deployment.yaml
```

Verify green deployment

```bash
kubectl get pods -l version=green
```

Test green environment directly

```bash
kubectl port-forward deployment/webapp-green 8080:80
curl http://localhost:8080
```

### **Step 5: Switch Traffic to Green**

Update service to point to green environment

```bash
kubectl patch service webapp-service -p '{"spec":{"selector":{"version":"green"}}}'
```

Verify traffic switch

```bash
curl http://<service-external-ip>
```

Monitor the switch

```bash
kubectl get endpoints webapp-service -w
```

### **Step 6: Rollback to Blue (if needed)**

Rollback to blue environment

```bash
kubectl patch service webapp-service -p '{"spec":{"selector":{"version":"blue"}}}'
```

Verify rollback

```bash
curl http://<service-external-ip>
```

### **Step 7: Cleanup**

After successful deployment, remove old environment

```bash
kubectl delete deployment webapp-blue
```

Keep green as the new production environment
