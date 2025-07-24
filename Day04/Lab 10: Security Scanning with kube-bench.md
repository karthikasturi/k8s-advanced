### **Lab 10: Security Scanning with kube-bench**

**Scenario**: Use kube-bench to audit cluster security against CIS benchmarks.

**Step 1**: Run kube-bench

```bash
# Run kube-bench as a job
kubectl apply -f https://raw.githubusercontent.com/aquasecurity/kube-bench/main/job.yaml

# Wait for completion and check results
kubectl logs job/kube-bench

# Or run kube-bench directly if installed locally
# kube-bench run --targets node,policies,controlplane,etcd
```

**Step 2**: Analyze results and remediate findings

```bash
# Example remediation for common findings:

# 1. Ensure that the --anonymous-auth argument is set to false
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --anonymous-auth=false

# 2. Ensure that the --insecure-port argument is set to 0
# Add: --insecure-port=0

# 3. Ensure that the kubelet --anonymous-auth argument is set to false
# Edit /var/lib/kubelet/config.yaml
# Set: authentication.anonymous.enabled: false
```
