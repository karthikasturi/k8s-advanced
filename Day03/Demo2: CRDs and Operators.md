## Demo 2: Custom Resource Definitions (CRDs) and Operators

### 2.1: Creating and Managing CRDs

**Create database-crd.yaml:**

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.example.com
spec:
  group: example.com
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
              engine:
                type: string
                enum: ["postgresql", "mysql", "mongodb"]
              version:
                type: string
              replicas:
                type: integer
                minimum: 1
                maximum: 5
              storage:
                type: string
                pattern: '^[0-9]+[GM]i$'
              backup:
                type: object
                properties:
                  enabled:
                    type: boolean
                  schedule:
                    type: string
            required:
            - engine
            - version
          status:
            type: object
            properties:
              phase:
                type: string
                enum: ["Pending", "Running", "Failed"]
              message:
                type: string
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
```

```bash
# Apply CRD
kubectl apply -f database-crd.yaml

# Verify CRD
kubectl get crd
kubectl describe crd databases.example.com
```

### 2.2: Creating Custom Resources

**Create postgres-instance.yaml:**

```yaml
apiVersion: example.com/v1
kind: Database
metadata:
  name: my-postgres
  namespace: default
spec:
  engine: postgresql
  version: "13"
  replicas: 2
  storage: "10Gi"
  backup:
    enabled: true
    schedule: "0 2 * * *"
```

```bash
# Create custom resource
kubectl apply -f postgres-instance.yaml

# View custom resources
kubectl get databases
kubectl get db
kubectl describe database my-postgres

# Show resource in YAML
kubectl get database my-postgres -o yaml
```

### 2.3: Simple Operator Simulation

**Create simple-controller.py:**

```python
#!/usr/bin/env python3
import time
import yaml
from kubernetes import client, config, watch

def create_deployment_for_database(db_name, db_spec, namespace):
    """Create a Deployment based on Database CR"""
    deployment = {
        'apiVersion': 'apps/v1',
        'kind': 'Deployment',
        'metadata': {
            'name': f'{db_name}-deployment',
            'namespace': namespace,
            'labels': {'managed-by': 'database-operator'}
        },
        'spec': {
            'replicas': db_spec.get('replicas', 1),
            'selector': {'matchLabels': {'app': db_name}},
            'template': {
                'metadata': {'labels': {'app': db_name}},
                'spec': {
                    'containers': [{
                        'name': db_spec['engine'],
                        'image': f"{db_spec['engine']}:{db_spec['version']}",
                        'env': [{'name': 'POSTGRES_PASSWORD', 'value': 'password'}]
                    }]
                }
            }
        }
    }
    return deployment

def watch_database_resources():
    """Watch for Database resource changes"""
    config.load_kube_config()
    v1 = client.ApiClient()

    # Watch custom resources
    w = watch.Watch()
    for event in w.stream(client.CustomObjectsApi().list_cluster_custom_object,
                         group="example.com", version="v1", plural="databases"):
        obj = event['object']
        event_type = event['type']
        name = obj['metadata']['name']
        namespace = obj['metadata']['namespace']

        print(f"Event: {event_type} - Database: {name}")

        if event_type == 'ADDED':
            print(f"Creating deployment for {name}")
            # Here you would create the actual deployment
            # deployment = create_deployment_for_database(name, obj['spec'], namespace)

if __name__ == "__main__":
    watch_database_resources()
```
