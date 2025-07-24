# **Clean Up Resources**

```bash
# Remove test resources
kubectl delete namespace development monitoring frontend backend database
kubectl delete namespace production-web production-api production-db

# Remove Gatekeeper constraints
kubectl delete AllowedRegistries --all
kubectl delete ResourceLimits --all

# Remove constraint templates
kubectl delete constrainttemplate allowedregistries resourcelimits

# Uninstall Gatekeeper (optional)
kubectl delete -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# Uninstall cert-manager (optional)
kubectl delete -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml
```


