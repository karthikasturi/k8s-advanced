### **Lab 4: Implementing Network Policies**

**Scenario**: Implement a three-tier architecture with proper network segmentation.

**Step 0**: Create namespaces, pods and services

```bash
# Deploy test applications
kubectl create namespace frontend
kubectl create namespace backend
kubectl create namespace database

# Deploy nginx pods in each namespace
kubectl run web --image=nginx -n frontend
kubectl run api1 --image=nginx -n backend  
kubectl run db --image=nginx -n database

# Create services for each pod
kubectl expose pod web --port=80 --target-port=80 --name=web -n frontend
kubectl expose pod api1 --port=80 --target-port=80 --name=api -n backend
kubectl expose pod db --port=80 --target-port=80 --name=db -n database
```

**Step 1**: Create default deny-all policy

```yaml
# default-deny.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
```

```bash
# Apply to all namespaces
kubectl apply -f default-deny.yaml -n frontend
kubectl apply -f default-deny.yaml -n backend
kubectl apply -f default-deny.yaml -n database
```

**Step 2**: Verify connectivity is blocked

```bash
# These should now timeout/fail
kubectl exec -n frontend web -- curl -s --connect-timeout 5 http://api.backend.svc.cluster.local
kubectl exec -n backend api1 -- curl -s --connect-timeout 5 http://db.database.svc.cluster.local
```

**Step 3**: Allow frontend to backend communication

```yaml
# frontend-to-backend.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: backend
spec:
  podSelector:
    matchLabels:
      run: api1
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: frontend
    ports:
    - protocol: TCP
      port: 80
```

```bash
# Label frontend namespace for selection
kubectl label namespace frontend name=frontend

kubectl apply -f frontend-to-backend.yaml
```

**Step 4**: Allow backend to database communication

```yaml
# backend-to-database.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-backend-to-database
  namespace: database
spec:
  podSelector:
    matchLabels:
      run: db
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: backend
    ports:
    - protocol: TCP
      port: 80
```

```bash
# Label backend namespace
kubectl label namespace backend name=backend

kubectl apply -f backend-to-database.yaml
```

**Step 5**: Allow egress for DNS resolution

```yaml
# allow-dns.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to: []
    ports:
    - protocol: UDP
      port: 53
    - protocol: TCP
      port: 53
```

```bash
# Apply to all namespaces
kubectl apply -f allow-dns.yaml -n frontend
kubectl apply -f allow-dns.yaml -n backend
kubectl apply -f allow-dns.yaml -n database
```

**Step 6**: Test the segmented network

```bash
# This should work: frontend -> backend
kubectl exec -n frontend web -- curl -s http://api.backend.svc.cluster.local

# This should work: backend -> database  
kubectl exec -n backend api -- curl -s http://db.database.svc.cluster.local

# This should be blocked: frontend -> database
kubectl exec -n frontend web -- curl -s --connect-timeout 5 http://db.database.svc.cluster.local
```
