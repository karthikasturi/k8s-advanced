## **RBAC and Pod Security Labs**

### **Demo 1: Understanding Default Permissions**

**Objective**: Explore default cluster permissions and identify security gaps

```bash
# Check current user permissions
kubectl auth can-i "*" "*"
kubectl auth can-i create pods
kubectl auth can-i delete nodes

# List all cluster roles
kubectl get clusterroles

# Examine a high-privilege role
kubectl describe clusterrole cluster-admin

# Check default service account permissions
kubectl auth can-i create pods --as=system:serviceaccount:default:default
```

### **Demo 2: Pod Security Standards Implementation**

**Objective**: Demonstrate the three security profiles and their enforcement

```bash
# Create namespaces for different security levels
kubectl create namespace privileged-apps
kubectl create namespace baseline-apps  
kubectl create namespace restricted-apps

# Apply Pod Security Standards labels
kubectl label namespace privileged-apps pod-security.kubernetes.io/enforce=privileged
kubectl label namespace baseline-apps pod-security.kubernetes.io/enforce=baseline
kubectl label namespace restricted-apps pod-security.kubernetes.io/enforce=restricted

# Enable warnings and auditing
kubectl label namespace restricted-apps pod-security.kubernetes.io/warn=restricted
kubectl label namespace restricted-apps pod-security.kubernetes.io/audit=restricted
```

#### Test Policy Enforcement

```bash
# privileged
kubectl apply -n privileged-apps -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: privileged-test
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
```

```bash
# baseline
kubectl apply -n baseline-apps -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: baseline-test
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
```

```bash
# restricted compliant pod
kubectl apply -n restricted-apps -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restricted-compliant
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: test
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
EOF
```

```bash
# restricted non-compliant pod
kubectl apply -n restricted-apps -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: restricted-violation
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      runAsUser: 0
EOF
```

## **Network Policies and Secrets Management **

### **Demo 3: Default Network Behavior**

**Objective**: Demonstrate default flat network model and security implications

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

# Test connectivity (should work after services are created)
kubectl exec -n frontend web -- curl -s http://api.backend.svc.cluster.local
kubectl exec -n frontend web -- curl -s http://db.database.svc.cluster.local
kubectl exec -n backend api1 -- curl -s http://db.database.svc.cluster.local
```

### **Demo 4: Secrets Management**

**Objective**: Demonstrate different ways to create and use secrets

```bash
# Create secret from command line
kubectl create secret generic app-config \
  --from-literal=database-url="postgresql://user:pass@localhost/db" \
  --from-literal=api-key="abc123def456"

# Create secret from file
echo -n "supersecretpassword" > password.txt
kubectl create secret generic file-secret --from-file=password=password.txt
rm password.txt

# Examine the secret (note base64 encoding)
kubectl get secret app-config -o yaml
kubectl get secret app-config -o jsonpath='{.data.database-url}' | base64 -d
```

## **Image Scanning, TLS, and Admission Controllers **

### **Demo 5: Image Vulnerability Scanning**

**Objective**: Demonstrate image scanning using Trivy

```bash
# Install Trivy (if not available)
# On Ubuntu/Debian:
# curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scan a container image for vulnerabilities
trivy image nginx:latest

# Scan for specific severity levels
trivy image --severity HIGH,CRITICAL nginx:latest

# Scan for secrets in image
trivy image --scanners secret nginx:latest

# Generate JSON report
trivy image --format json --output report.json nginx:latest
```

### **Demo 6: TLS Certificate Management**

**Objective**: Examine cluster certificates and implement cert-manager

```bash
# Check API server certificates
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 5 "Subject:"

# Check certificate expiration
sudo openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# List all certificates in pki directory
sudo ls -la /etc/kubernetes/pki/

# Check etcd certificates
sudo openssl x509 -in /etc/kubernetes/pki/etcd/server.crt -text -noout | grep -A 5 "Subject:"
```
