### **Lab 7: Automated Certificate Management**

**Scenario**: Install and configure cert-manager for automatic certificate management.

**Step 1**: Install cert-manager

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl wait --for=condition=Ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=90s
```

**Step 2**: Create a self-signed ClusterIssuer

```yaml
# selfsigned-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

```bash
kubectl apply -f selfsigned-issuer.yaml
```

**Step 3**: Create a Certificate resource

```yaml
# app-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: app-tls-cert
  namespace: default
spec:
  secretName: app-tls-managed
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer
  dnsNames:
  - app.example.com
  - api.example.com
```

```bash
kubectl apply -f app-certificate.yaml

# Check certificate status
kubectl describe certificate app-tls-cert

# Verify secret was created
kubectl get secret app-tls-managed
```
