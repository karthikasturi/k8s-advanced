# LAB 2: Creating and Managing CRDs

**Objective**: Create Custom Resource Definitions and manage custom resources.

## Lab Setup

```bash
# Create lab directory
mkdir -p ~/k8s-labs/day3/crd-lab
cd ~/k8s-labs/day3/crd-lab
```

## Task 1: Create Application CRD

**Create application-crd.yaml:**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: applications.platform.example.com
spec:
  group: platform.example.com
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              name:
                type: string
                description: "Application name"
              image:
                type: string
                description: "Container image"
                pattern: '^[a-zA-Z0-9._/-]+:[a-zA-Z0-9._-]+$'
              replicas:
                type: integer
                minimum: 1
                maximum: 10
                default: 1
              port:
                type: integer
                minimum: 1
                maximum: 65535
                default: 8080
              environment:
                type: object
                additionalProperties:
                  type: string
              resources:
                type: object
                properties:
                  requests:
                    type: object
                    properties:
                      cpu:
                        type: string
                        pattern: '^[0-9]+m?$'
                      memory:
                        type: string
                        pattern: '^[0-9]+[KMGT]i?$'
                  limits:
                    type: object
                    properties:
                      cpu:
                        type: string
                        pattern: '^[0-9]+m?$'
                      memory:
                        type: string
                        pattern: '^[0-9]+[KMGT]i?$'
              ingress:
                type: object
                properties:
                  enabled:
                    type: boolean
                    default: false
                  host:
                    type: string
                  path:
                    type: string
                    default: "/"
            required:
            - name
            - image
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Running", "Failed", "Succeeded"]
              message:
                type: string
              replicas:
                type: integer
              readyReplicas:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                      enum: ["True", "False", "Unknown"]
                    lastTransitionTime:
                      type: string
                      format: date-time
                    reason:
                      type: string
                    message:
                      type: string
    additionalPrinterColumns:
    - name: Image
      type: string
      jsonPath: .spec.image
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
  scope: Namespaced
  names:
    plural: applications
    singular: application
    kind: Application
    shortNames:
    - app
    categories:
    - platform
```

```bash
# Apply CRD
kubectl apply -f application-crd.yaml

# Verify CRD
kubectl get crd applications.platform.example.com
kubectl describe crd applications.platform.example.com

# Check API resources
kubectl api-resources | grep applications
```

## Task 2: Create Application Instances

**Create web-app.yaml:**

```yaml
apiVersion: platform.example.com/v1
kind: Application
metadata:
  name: web-app
  namespace: default
spec:
  name: "web-application"
  image: "nginx:1.21"
  replicas: 3
  port: 80
  environment:
    ENV: "production"
    DEBUG: "false"
    APP_NAME: "web-app"
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  ingress:
    enabled: true
    host: "web-app.local"
    path: "/"
```

**Create api-app.yaml:**

```yaml
apiVersion: platform.example.com/v1
kind: Application
metadata:
  name: api-app
  namespace: default
spec:
  name: "api-service"
  image: "node:16-alpine"
  replicas: 2
  port: 3000
  environment:
    NODE_ENV: "production"
    PORT: "3000"
    DATABASE_URL: "postgresql://user:pass@db:5432/mydb"
  resources:
    requests:
      cpu: "200m"
      memory: "256Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  ingress:
    enabled: true
    host: "api.local"
    path: "/api"
```

```bash
# Create application instances
kubectl apply -f web-app.yaml
kubectl apply -f api-app.yaml

# List applications
kubectl get applications
kubectl get app

# Describe applications
kubectl describe application web-app
kubectl describe application api-app

# Get applications in different formats
kubectl get application web-app -o yaml
kubectl get application api-app -o json
```

## Task 3: Update Application Status

**Create status-update.yaml:**

```yaml
apiVersion: platform.example.com/v1
kind: Application
metadata:
  name: web-app
  namespace: default
spec:
  name: "web-application"
  image: "nginx:1.21"
  replicas: 3
  port: 80
  environment:
    ENV: "production"
    DEBUG: "false"
    APP_NAME: "web-app"
  resources:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "500m"
      memory: "512Mi"
  ingress:
    enabled: true
    host: "web-app.local"
    path: "/"
status:
  phase: "Running"
  message: "Application is running successfully"
  replicas: 3
  readyReplicas: 3
  conditions:
  - type: "Ready"
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:00Z"
    reason: "ApplicationReady"
    message: "All replicas are ready"
  - type: "Available"
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:00Z"
    reason: "ApplicationAvailable"
    message: "Application is available"
```

```bash
# Update application status (simulating operator behavior)
kubectl apply -f status-update.yaml

# Check updated status
kubectl get applications
kubectl describe application web-app

# Create simple script to simulate controller behavior
cat > simple-controller.sh << 'EOF'
#!/bin/bash

echo "Simple Application Controller - Monitoring applications..."

while true; do
    # Get all applications
    apps=$(kubectl get applications -o name 2>/dev/null)

    for app in $apps; do
        app_name=$(echo $app | cut -d'/' -f2)
        echo "Processing application: $app_name"

        # Get application spec
        replicas=$(kubectl get $app -o jsonpath='{.spec.replicas}')
        image=$(kubectl get $app -o jsonpath='{.spec.image}')

        echo "  - Desired replicas: $replicas"
        echo "  - Image: $image"
        echo "  - Status: Would create/update Deployment here"

        # In a real operator, you would:
        # 1. Create/update Deployment based on Application spec
        # 2. Create/update Service if needed
        # 3. Create/update Ingress if enabled
        # 4. Update Application status

    done

    echo "---"
    sleep 30
done
EOF

chmod +x simple-controller.sh

# Run controller in background for demonstration
./simple-controller.sh &
CONTROLLER_PID=$!

# Let it run for a minute
sleep 60

# Stop the controller
kill $CONTROLLER_PID
```
