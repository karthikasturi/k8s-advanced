# **Final Integration Lab**

### **Lab 11: Complete Security Implementation**

**Scenario**: Implement a complete security setup for a multi-tier application.

**Step 1**: Create secure namespaces

```bash
# Create namespaces with Pod Security Standards
kubectl create namespace production-web
kubectl create namespace production-api  
kubectl create namespace production-db

# Apply restrictive Pod Security Standards
for ns in production-web production-api production-db; do
  kubectl label namespace $ns pod-security.kubernetes.io/enforce=restricted
  kubectl label namespace $ns pod-security.kubernetes.io/audit=restricted
  kubectl label namespace $ns pod-security.kubernetes.io/warn=restricted
  kubectl label namespace $ns name=$ns
done
```

**Step 2**: Implement RBAC

```bash
# Create service accounts for each tier
kubectl create serviceaccount web-sa -n production-web
kubectl create serviceaccount api-sa -n production-api
kubectl create serviceaccount db-sa -n production-db

# Apply minimal RBAC roles (create roles and bindings as in previous labs)
```

**Step 3**: Deploy applications with security hardening

```yaml
# production-web-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
      tier: frontend
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      serviceAccountName: web-sa
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: web
        image: nginx:alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        resources:
          limits:
            cpu: "500m"
            memory: "512Mi"
          requests:
            cpu: "250m"
            memory: "256Mi"
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
        - name: run
          mountPath: /var/run
      volumes:
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
      - name: run
        emptyDir: {}
```

**Step 4**: Implement Network Policies

```bash
# Apply default deny and specific allow policies as demonstrated in Lab 4
```

**Step 5**: Enable monitoring and alerting

```bash
# Deploy Falco for runtime security monitoring
kubectl apply -f https://raw.githubusercontent.com/falcosecurity/deploy-kubernetes/main/kubernetes/falco/templates/daemonset.yaml

# Check Falco logs for security events
kubectl logs -l app.kubernetes.io/name=falco -n falco --tail=100
```

## **Verification and Testing**

### **Security Validation Checklist**

Run these commands to verify your security implementation:

```bash
# 1. Verify RBAC is working
kubectl auth can-i create pods --as=system:serviceaccount:production-web:web-sa -n production-api
# Should return "no"

# 2. Test Network Policies
kubectl exec -n production-web deployment/web-app -- curl -s --connect-timeout 5 http://db-service.production-db.svc.cluster.local
# Should timeout/fail if policies are correct

# 3. Verify Pod Security Standards
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
  namespace: production-web
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
# Should be rejected

# 4. Check Gatekeeper constraints
kubectl get constraints

# 5. Verify secrets are not exposed
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].env[*]}' | grep -i secret
# Should not return plain text secrets

# 6. Test image registry policy
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: untrusted-registry-test
spec:
  containers:
  - name: test
    image: untrusted.registry.com/malicious:latest
EOF
# Should be rejected by Gatekeeper
```
