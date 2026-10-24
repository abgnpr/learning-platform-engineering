```
kubectl apply -f secret.yaml -f storage.yaml -f deployment.yaml -f service.yaml
```

```
kubectl port-forward svc/postgres-service 5432:5432
```

```
minikube service postgres-service --url
```
