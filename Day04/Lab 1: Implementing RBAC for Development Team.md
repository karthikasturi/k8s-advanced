### **Lab 1: Implementing RBAC for Development Team**

**Scenario**: Create RBAC policies for a development team that needs limited access to a specific namespace.

**Step 1**: Create namespace and service account

```bash
# Create development namespace
kubectl create namespace development

# Create service account for dev team
kubectl create serviceaccount dev-team-sa -n development
```

**Step 2**: Create a Role with limited permissions

```yaml
# dev-role.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: dev-team-role
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets"]
  verbs: ["get", "list", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list"]  # Read-only access to secrets
```

```bash
kubectl apply -f dev-role.yaml
```

**Step 3**: Create RoleBinding

```yaml
# dev-rolebinding.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dev-team-binding
  namespace: development
subjects:
- kind: ServiceAccount
  name: dev-team-sa
  namespace: development
roleRef:
  kind: Role
  name: dev-team-role
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl apply -f dev-rolebinding.yaml
```

**Step 4**: Test permissions

```bash
# Test allowed operations
kubectl auth can-i create pods --as=system:serviceaccount:development:dev-team-sa -n development

# Test forbidden operations  
kubectl auth can-i delete secrets --as=system:serviceaccount:development:dev-team-sa -n development
kubectl auth can-i create pods --as=system:serviceaccount:development:dev-team-sa -n kube-system
```
