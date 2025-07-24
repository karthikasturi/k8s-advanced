### **Lab 2: ClusterRole for Monitoring Service**

**Scenario**: Create a ClusterRole for a monitoring service that needs read access across all namespaces.

**Step 1**: Create monitoring namespace and service account

```bash
kubectl create namespace monitoring
kubectl create serviceaccount prometheus-sa -n monitoring
```

**Step 2**: Create ClusterRole

```yaml
# monitoring-clusterrole.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: prometheus-reader
rules:
- apiGroups: [""]
  resources: ["nodes", "pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "daemonsets", "statefulsets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes/metrics", "pods/metrics"]
  verbs: ["get"]
- nonResourceURLs: ["/metrics"]
  verbs: ["get"]
```

```bash
kubectl apply -f monitoring-clusterrole.yaml
```

**Step 3**: Create ClusterRoleBinding

```yaml
# monitoring-clusterrolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: prometheus-binding
subjects:
- kind: ServiceAccount
  name: prometheus-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: prometheus-reader
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f monitoring-clusterrolebinding.yaml
```

**Step 4**: Verification

```bash
# Test cross-namespace read access
kubectl auth can-i get pods --as=system:serviceaccount:monitoring:prometheus-sa --all-namespaces

# Verify no write permissions
kubectl auth can-i create pods --as=system:serviceaccount:monitoring:prometheus-sa
```
