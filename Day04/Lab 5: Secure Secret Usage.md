### **Lab 5: Secure Secret Usage**

**Scenario**: Deploy an application that securely uses secrets without exposing them.

**Step 1**: Create TLS secret for web application

```bash
# Generate self-signed certificate for demo
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=app.example.com"

# Create TLS secret
kubectl create secret tls app-tls-secret --cert=tls.crt --key=tls.key

# Clean up certificate files
rm tls.key tls.crt
```

**Step 2**: Create application secrets

```yaml
# app-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: default
type: Opaque
data:
  db-username: YXBwdXNlcg==  # appuser
  db-password: c2VjdXJlcGFzcw==  # securepass
  jwt-secret: bXlzZWNyZXRqd3R0b2tlbg==  # mysecretjwttoken
```

```bash
kubectl apply -f app-secrets.yaml
```

**Step 3**: Deploy application using secrets

```yaml
# secure-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: secure-app
  template:
    metadata:
      labels:
        app: secure-app
    spec:
      serviceAccountName: default
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
      containers:
      - name: app
        image: nginx:alpine
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: db-username
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: jwt-secret
        volumeMounts:
        - name: db-password
          mountPath: "/etc/secrets"
          readOnly: true
        - name: tls-certs
          mountPath: "/etc/ssl/certs"
          readOnly: true
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache/nginx
      volumes:
      - name: db-password
        secret:
          secretName: app-secrets
          items:
          - key: db-password
            path: db-password
      - name: tls-certs
        secret:
          secretName: app-tls-secret
      - name: tmp
        emptyDir: {}
      - name: cache
        emptyDir: {}
```

```bash
kubectl apply -f secure-app.yaml
```

**Step 4**: Verify secret usage

```bash
# Check environment variables (DB_USERNAME and JWT_SECRET should be visible)
kubectl exec deployment/secure-app -- env | grep -E "(DB_|JWT_)"

# Check mounted secret file
kubectl exec deployment/secure-app -- cat /etc/secrets/db-password

# Check TLS certificates
kubectl exec deployment/secure-app -- ls -la /etc/ssl/certs/
```
