#### Deploy
```
kubectl apply -k .
```

#### Get token
```
kubectl describe secret -n kubernetes-dashboard admin-user-token-<hoge>
```

#### Access
```
kubectl proxy
```

@AnotherTerminal
```
open http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/overview?namespace=kubernetes-dashboard
```
