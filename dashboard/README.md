#### Deploy
```
kubectl apply -k .
```

#### Access
```
kubectl proxy
```

@AnotherTerminal
```
open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=kubernetes-dashboard
```
