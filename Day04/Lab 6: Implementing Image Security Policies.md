### **Lab 6: Implementing Image Security Policies**

**Scenario**: Create admission controller policies to prevent vulnerable images from being deployed.

**Step 1**: Install OPA Gatekeeper

```bash
# Install Gatekeeper
kubectl apply -f https://raw.githubusercontent.com/open-policy-agent/gatekeeper/release-3.14/deploy/gatekeeper.yaml

# Wait for Gatekeeper to be ready
kubectl wait --for=condition=Ready pod -l control-plane=controller-manager -n gatekeeper-system --timeout=90s
```

**Step 2**: Create constraint template for allowed registries

```yaml
# allowed-registries-template.yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: allowedregistries
spec:
  crd:
    spec:
      names:
        kind: AllowedRegistries
      validation:
        type: object
        properties:
          registries:
            type: array
            items:
              type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package allowedregistries

        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not starts_with(container.image, input.parameters.registries[_])
          msg := sprintf("Container image '%v' is not from an allowed registry", [container.image])
        }

        violation[{"msg": msg}] {
          container := input.review.object.spec.initContainers[_]
          not starts_with(container.image, input.parameters.registries[_])
          msg := sprintf("Init container image '%v' is not from an allowed registry", [container.image])
        }
```

```bash
kubectl apply -f allowed-registries-template.yaml
```

**Step 3**: Create constraint to enforce policy

```yaml
# registry-constraint.yaml
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedRegistries
metadata:
  name: must-use-trusted-registries
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    registries:
      - "docker.io/"
      - "gcr.io/"
      - "registry.k8s.io/"
      - "quay.io/"
```

```bash
kubectl apply -f registry-constraint.yaml
```

**Step 4**: Test the policy

```yaml
# bad-image-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: bad-image-pod
spec:
  containers:
  - name: app
    image: malicious-registry.com/bad-image:latest
```

```bash
# This should be rejected
kubectl apply -f bad-image-pod.yaml
```

```yaml
# good-image-pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: good-image-pod
spec:
  containers:
  - name: app
    image: docker.io/nginx:alpine
```

```bash
# This should be accepted
kubectl apply -f good-image-pod.yaml
```
