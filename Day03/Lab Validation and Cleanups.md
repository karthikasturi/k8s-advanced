## Lab Validation and Cleanup

```bash
# Validate all deployments
echo "=== Helm Deployments ==="
helm list --all-namespaces

echo "=== Custom Resources ==="
kubectl get applications

echo "=== StatefulSets ==="
kubectl get statefulsets

echo "=== DaemonSets ==="
kubectl get daemonsets --all-namespaces

echo "=== Persistent Volumes ==="
kubectl get pvc
kubectl get pv

# Cleanup (optional)
echo "Cleaning up lab resources..."

# Clean up Helm releases
helm uninstall my-wordpress -n wordpress
helm uninstall myapp
kubectl delete namespace wordpress

# Clean up CRDs and custom resources
kubectl delete applications --all
kubectl delete crd applications.platform.example.com

# Clean up StatefulSets
kubectl delete statefulset mongodb
kubectl delete svc mongodb mongodb-headless
kubectl delete configmap mongodb-config
kubectl delete secret mongodb-secret
kubectl delete pvc -l app=mongodb

# Clean up DaemonSets
kubectl delete daemonset log-collector -n kube-system
kubectl delete configmap log-collector-config -n kube-system
kubectl delete clusterrolebinding log-collector
kubectl delete clusterrole log-collector
kubectl delete serviceaccount log-collector -n kube-system

echo "Lab cleanup completed!"
```

## Lab Summary

### Key Achievements

1. **Helm Mastery**: Deployed applications using existing charts and created custom charts with templates
2. **CRD Implementation**: Created custom resource definitions and managed custom resources
3. **StatefulSet Deployment**: Set up MongoDB cluster with persistent storage and stable network identities
4. **DaemonSet Configuration**: Implemented log collection across all cluster nodes

### Best Practices Learned

- Helm chart organization and templating
- CRD validation and versioning
- StatefulSet storage management
- DaemonSet resource optimization
- Proper RBAC configuration
