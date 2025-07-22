## **Cleanup Commands**

Cleanup Blue/Green Demo

```bash
kubectl delete deployment webapp-blue webapp-green
kubectl delete service webapp-service
```

Cleanup Canary Demo

```bash
kubectl delete deployment webapp-v1 webapp-v2
```

Cleanup Rolling Update Demo

```bash
kubectl delete deployment rolling-demo advanced-rolling-demo
```

Cleanup HPA Lab

```bash
kubectl delete hpa webapp-hpa webapp-advanced-hpa webapp-custom-hpa
kubectl delete deployment webapp-hpa-demo
kubectl delete service webapp-hpa-service
kubectl delete pod load-generator --ignore-not-found
```

Cleanup VPA Lab

```bash
kubectl delete vpa vpa-demo-recommender vpa-demo-auto vpa-demo-initial
kubectl delete deployment vpa-demo vpa-demo-initial
```

Cleanup Combined Lab

```bash
kubectl delete hpa combined-demo-hpa
kubectl delete vpa combined-demo-vpa
kubectl delete deployment combined-demo
```

Cleanup Troubleshooting Lab

```bash
kubectl delete hpa broken-hpa
kubectl delete deployment broken-hpa-demo
```
