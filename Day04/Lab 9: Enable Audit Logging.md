### **Lab 9: Enable Audit Logging**

**Scenario**: Configure comprehensive audit logging for security monitoring.

**Step 1**: Create audit policy

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# Log authentication failures
- level: Metadata
  namespaces: [""]
  verbs: ["create"]
  resources:
  - group: ""
    resources: ["pods", "services"]

# Log all secret access
- level: RequestResponse
  resources:
  - group: ""
    resources: ["secrets"]

# Log RBAC changes
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "clusterroles", "rolebindings", "clusterrolebindings"]

# Log admission controller decisions
- level: Request
  users: ["system:serviceaccount:gatekeeper-system:gatekeeper-admin"]

# Don't log routine system operations
- level: None
  users: ["system:kube-proxy"]
  verbs: ["watch"]
  resources:
  - group: ""
    resources: ["endpoints", "services"]
```

**Step 2**: Apply audit configuration (requires cluster admin access)

```bash
# Note: This typically requires modifying the API server configuration
# For managed clusters, this may not be possible

# For kubeadm clusters, you would:
# 1. Place audit-policy.yaml in /etc/kubernetes/
# 2. Modify /etc/kubernetes/manifests/kube-apiserver.yaml to add:
#    --audit-log-path=/var/log/audit.log
#    --audit-policy-file=/etc/kubernetes/audit-policy.yaml
#    --audit-log-maxage=30
#    --audit-log-maxbackup=3
#    --audit-log-maxsize=100
```
