### **Lab 3: Pod Security Contexts**

**Scenario**: Deploy pods with different security contexts to understand restrictions.

**Step 1**: Create a privileged pod (will be rejected in restricted namespace)

```yaml
# privileged-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: app
    image: nginx
    securityContext:
      privileged: true
      allowPrivilegeEscalation: true
```

**Step 2**: Test deployment in different namespaces

```bash
# This should succeed
kubectl apply -f privileged-pod.yaml -n privileged-apps

# This should be rejected
kubectl apply -f privileged-pod.yaml -n restricted-apps
```

**Step 3**: Create a secure pod

```yaml
# secure-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:alpine
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      runAsNonRoot: true
      capabilities:
        drop:
        - ALL
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

```bash
# This should succeed in all namespaces
kubectl apply -f secure-pod.yaml -n restricted-apps
kubectl apply -f secure-pod.yaml -n baseline-apps
```
