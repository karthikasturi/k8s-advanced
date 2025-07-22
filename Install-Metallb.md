# <Unverified>

## 1. MetalLB (Most Popular)

MetalLB provides LoadBalancer functionality for bare-metal and local Kubernetes clusters[](https://stackoverflow.com/questions/59760559/kubernetes-default-loadbalancer-for-a-service-run-locally)[](https://dzone.com/articles/how-to-create-a-kubernetes-cluster-and-load-balanc):

```bash
# Install MetalLB
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.7/config/manifests/metallb-native.yaml
```

```yaml
# Configure IP pool
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.1.240-192.168.1.250
```

**Benefits:**

- Works with kubeadm clusters

- Assigns real external IPs from configured pool

- Supports both Layer 2 and BGP modes

- Production-ready solution


