

# LAB 3: StatefulSets and DaemonSets

**Objective**: Deploy and manage StatefulSets and DaemonSets for different use cases.

## Lab Setup

```bash
# Create lab directory
mkdir -p ~/k8s-labs/day3/stateful-lab
cd ~/k8s-labs/day3/stateful-lab
```

## Task 1: Deploy MongoDB StatefulSet

**Create storage-class.yaml:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**Create mongodb-configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-config
data:
  mongod.conf: |
    storage:
      dbPath: /data/db
      journal:
        enabled: true
    systemLog:
      destination: file
      logAppend: true
      path: /var/log/mongodb/mongod.log
      quiet: false
      verbosity: 0
    net:
      port: 27017
      bindIp: 0.0.0.0
    replication:
      replSetName: rs0
    security:
      authorization: disabled
  init-replica.js: |
    rs.initiate({
      _id: "rs0",
      members: [
        { _id: 0, host: "mongodb-0.mongodb-headless:27017" },
        { _id: 1, host: "mongodb-1.mongodb-headless:27017" },
        { _id: 2, host: "mongodb-2.mongodb-headless:27017" }
      ]
    });
```

**Create mongodb-secret.yaml:**

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  # admin:password (base64 encoded)
  username: YWRtaW4=
  password: cGFzc3dvcmQ=
```

**Create mongodb-headless-service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb-headless
  labels:
    app: mongodb
spec:
  clusterIP: None
  ports:
  - port: 27017
    targetPort: 27017
    name: mongodb
  selector:
    app: mongodb
```

**Create mongodb-service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  labels:
    app: mongodb
spec:
  ports:
  - port: 27017
    targetPort: 27017
    name: mongodb
  selector:
    app: mongodb
```

**Create mongodb-statefulset.yaml:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
spec:
  serviceName: "mongodb-headless"
  replicas: 3
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo:5.0
        command:
        - mongod
        - --config
        - /etc/mongod.conf
        ports:
        - containerPort: 27017
          name: mongodb
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: username
        - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: password
        volumeMounts:
        - name: mongodb-data
          mountPath: /data/db
        - name: config
          mountPath: /etc/mongod.conf
          subPath: mongod.conf
        - name: init-script
          mountPath: /docker-entrypoint-initdb.d/init-replica.js
          subPath: init-replica.js
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          exec:
            command:
            - mongosh
            - --eval
            - "db.adminCommand('ping')"
          initialDelaySeconds: 5
          periodSeconds: 5
      volumes:
      - name: config
        configMap:
          name: mongodb-config
          items:
          - key: mongod.conf
            path: mongod.conf
      - name: init-script
        configMap:
          name: mongodb-config
          items:
          - key: init-replica.js
            path: init-replica.js
  volumeClaimTemplates:
  - metadata:
      name: mongodb-data
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 2Gi
```

```bash
# Apply all MongoDB resources
kubectl apply -f storage-class.yaml
kubectl apply -f mongodb-secret.yaml
kubectl apply -f mongodb-configmap.yaml
kubectl apply -f mongodb-headless-service.yaml
kubectl apply -f mongodb-service.yaml
kubectl apply -f mongodb-statefulset.yaml

# Watch StatefulSet deployment
kubectl get pods -w -l app=mongodb

# Check persistent volumes
kubectl get pvc
kubectl get pv

# Test MongoDB connectivity
kubectl run mongodb-client --rm -it --restart=Never --image=mongo:5.0 -- bash
# Inside the pod:
mongosh --host mongodb-0.mongodb-headless:27017
# Try some commands:
# show dbs
# use testdb
# db.test.insert({name: "test", value: 123})
# db.test.find()
# exit
exit

# Scale StatefulSet
kubectl scale statefulset mongodb --replicas=5
kubectl get pods -w -l app=mongodb

# Rolling update
kubectl patch statefulset mongodb -p '{"spec":{"template":{"spec":{"containers":[{"name":"mongodb","image":"mongo:6.0"}]}}}}'
kubectl rollout status statefulset/mongodb
```

## Task 2: Create Log Collection DaemonSet

**Create log-collector-rbac.yaml:**

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: log-collector
  namespace: kube-system

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: log-collector
rules:
- apiGroups: [""]
  resources: ["nodes", "nodes/proxy", "services", "endpoints", "pods"]
  verbs: ["get", "list", "watch"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: log-collector
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: log-collector
subjects:
- kind: ServiceAccount
  name: log-collector
  namespace: kube-system
```

**Create log-collector-configmap.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: log-collector-config
  namespace: kube-system
data:
  fluent-bit.conf: |
    [SERVICE]
        Daemon Off
        Flush 1
        Log_Level info
        Parsers_File parsers.conf
        HTTP_Server On
        HTTP_Listen 0.0.0.0
        HTTP_Port 2020

    [INPUT]
        Name tail
        Path /var/log/containers/*.log
        Parser docker
        Tag kube.*
        Refresh_Interval 5
        Mem_Buf_Limit 5MB
        Skip_Long_Lines On

    [INPUT]
        Name systemd
        Tag host.*
        Systemd_Filter _SYSTEMD_UNIT=kubelet.service
        Read_From_Tail On

    [FILTER]
        Name kubernetes
        Match kube.*
        Kube_URL https://kubernetes.default.svc:443
        Kube_CA_File /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log On
        K8S-Logging.Parser On
        K8S-Logging.Exclude Off

    [OUTPUT]
        Name stdout
        Match *
        Format json_lines

  parsers.conf: |
    [PARSER]
        Name docker
        Format json
        Time_Key time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep On
```

**Create log-collector-daemonset.yaml:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: kube-system
  labels:
    app: log-collector
spec:
  selector:
    matchLabels:
      name: log-collector
  template:
    metadata:
      labels:
        name: log-collector
    spec:
      serviceAccountName: log-collector
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: log-collector
        image: fluent/fluent-bit:2.0
        resources:
          limits:
            memory: 200Mi
            cpu: 100m
          requests:
            memory: 100Mi
            cpu: 50m
        ports:
        - containerPort: 2020
          name: http-metrics
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config
          mountPath: /fluent-bit/etc/fluent-bit.conf
          subPath: fluent-bit.conf
        - name: config
          mountPath: /fluent-bit/etc/parsers.conf
          subPath: parsers.conf
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config
        configMap:
          name: log-collector-config
```

```bash
# Deploy log collector DaemonSet
kubectl apply -f log-collector-rbac.yaml
kubectl apply -f log-collector-configmap.yaml
kubectl apply -f log-collector-daemonset.yaml

# Check DaemonSet status
kubectl get daemonset -n kube-system log-collector
kubectl get pods -n kube-system -l name=log-collector -o wide

# View logs from log collector
kubectl logs -n kube-system -l name=log-collector --tail=20

# Check metrics endpoint
kubectl port-forward -n kube-system svc/log-collector 2020:2020 &
curl http://localhost:2020/api/v1/metrics/prometheus

# Update DaemonSet with resource changes
kubectl patch daemonset log-collector -n kube-system -p '{"spec":{"template":{"spec":{"containers":[{"name":"log-collector","resources":{"requests":{"memory":"150Mi"},"limits":{"memory":"300Mi"}}}]}}}}'

# Check rollout status
kubectl rollout status daemonset/log-collector -n kube-system
```
