### **Lab 8: Custom Admission Controller**

**Scenario**: Create a simple validating admission webhook that enforces resource limits.

### Install Gatekeeper(using Helm)

```bash
# Add the Gatekeeper Helm repository
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

# Install Gatekeeper
helm install gatekeeper gatekeeper/gatekeeper --namespace gatekeeper-system --create-namespace
```

#### Verify Installation

After installation, confirm that all components are working:

```bash
# Wait for all pods to be ready
kubectl wait --for=condition=Ready pod -l gatekeeper.sh/system=yes -n gatekeeper-system --timeout=300s

# Verify ConstraintTemplate CRD is created
kubectl get crd constrainttemplates.templates.gatekeeper.sh

# Check controller logs for any issues
kubectl logs -n gatekeeper-system deployment/gatekeeper-controller-manager
```

**Step 1**: Create constraint template for resource requirements

```yaml
# resource-limits-template.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: resourcelimits
spec:
  crd:
    spec:
      names:
        kind: ResourceLimits
      validation:
        openAPIV3Schema:
          type: object
          properties:
            maxCPU:
              type: string
            maxMemory:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package resourcelimits

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := "Container must have CPU limits set"
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := "Container must have memory limits set"
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          cpu_limit := container.resources.limits.cpu
          max_cpu := input.parameters.maxCPU
          cpu_limit > max_cpu
          msg := sprintf("CPU limit %v exceeds maximum allowed %v", [cpu_limit, max_cpu])
        }
```

```bash
kubectl apply -f resource-limits-template.yaml
```

**Step 2**: Create constraint

```yaml
# resource-limits-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: ResourceLimits
metadata:
  name: must-have-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    maxCPU: "2"
    maxMemory: "4Gi"
```

```bash
kubectl apply -f resource-limits-constraint.yaml
```

**Step 3**: Test the policy

```yaml
# no-limits-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: no-limits-pod
spec:  
  containers:
  - name: app
    image: nginx
    # No resource limits - should be rejected
```

```bash
# This should be rejected
kubectl apply -f no-limits-pod.yaml
```

```yaml
# proper-limits-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: proper-limits-pod
spec:
  containers:
  - name: app
    image: nginx
    resources:
      limits:
        cpu: "1"
        memory: "2Gi"
      requests:
        cpu: "500m"
        memory: "1Gi"
```

```bash
# This should be accepted
kubectl apply -f proper-limits-pod.yaml
```
