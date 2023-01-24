Refer to: https://www.rabbitmq.com/kubernetes/operator/quickstart-operator.html

## Steps:
```bash
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"

kubectl apply -f rabbitmq/rabbitmq.yaml
```

```
username="$(kubectl get secret production-ready-default-user -o jsonpath='{.data.username}' | base64 --decode)"
echo "username: $username"
password="$(kubectl get secret production-ready-default-user -o jsonpath='{.data.password}' | base64 --decode)"
echo "password: $password"

kubectl port-forward "service/production-ready" 15672
```