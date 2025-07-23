# Day 3: Kubernetes Advanced

## Demo 3: StatefulSets and DaemonSets

### 3.1: StatefulSet with Persistent Storage

**Create headless-service.yaml:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-headless
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: nginx
```

**Create statefulset.yaml:**

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx-headless"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
          name: web
        volumeMounts:
        - name: www
          mountPath: /usr/share/nginx/html
        - name: config
          mountPath: /etc/nginx/conf.d
        command:
        - sh
        - -c
        - |
          echo "Pod: $HOSTNAME" > /usr/share/nginx/html/index.html
          echo "Time: $(date)" >> /usr/share/nginx/html/index.html
          nginx -g 'daemon off;'
      volumes:
      - name: config
        configMap:
          name: nginx-config
  volumeClaimTemplates:
  - metadata:
      name: www
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
```

**Create nginx-config.yaml:**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  default.conf: |
    server {
        listen 80;
        location / {
            root /usr/share/nginx/html;
            index index.html;
        }
        location /health {
            access_log off;
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
```

```bash
# Deploy components
kubectl apply -f nginx-config.yaml
kubectl apply -f headless-service.yaml
kubectl apply -f statefulset.yaml

# Observe ordered creation
kubectl get pods -w

# Check persistent volumes
kubectl get pvc
kubectl get pv

# Test stable network identities
kubectl run -it --rm debug --image=busybox --restart=Never -- sh
# Inside the pod:
nslookup nginx-headless
nslookup web-0.nginx-headless
nslookup web-1.nginx-headless
exit

# Scale StatefulSet
kubectl scale statefulset web --replicas=5
kubectl get pods -w

# Rolling update
kubectl patch statefulset web -p '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.22"}]}}}}'
kubectl rollout status statefulset/web
```

### 3.2: DaemonSet for Node Monitoring

**Create node-monitor-daemonset.yaml:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  labels:
    app: node-monitor
spec:
  selector:
    matchLabels:
      name: node-monitor
  template:
    metadata:
      labels:
        name: node-monitor
    spec:
      tolerations:
      - key: node-role.kubernetes.io/master
        operator: Exists
        effect: NoSchedule
      containers:
      - name: node-monitor
        image: busybox:1.35
        resources:
          limits:
            memory: 64Mi
            cpu: 100m
          requests:
            memory: 32Mi
            cpu: 50m
        command:
        - sh
        - -c
        - |
          while true; do
            echo "$(date): Monitoring node $NODE_NAME"
            echo "Disk usage:"
            df -h
            echo "Memory usage:"
            free -h
            echo "Load average:"
            uptime
            echo "---"
            sleep 60
          done
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
      hostNetwork: true
      hostPID: true
```

```bash
# Deploy DaemonSet
kubectl apply -f node-monitor-daemonset.yaml

# Check DaemonSet status
kubectl get daemonset
kubectl get pods -o wide -l name=node-monitor

# View logs from different nodes
kubectl logs -l name=node-monitor --tail=10

# Update DaemonSet
kubectl patch daemonset node-monitor -p '{"spec":{"template":{"spec":{"containers":[{"name":"node-monitor","image":"busybox:1.36"}]}}}}'
kubectl rollout status daemonset/node-monitor
```
