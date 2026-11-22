curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash

helm version

helm install myapp ./test \
  --namespace helm-demo \
  --create-namespace

kubectl port-forward svc/myapp-web 8080:80 -n helm-demo
curl http://localhost:8080

helm upgrade myapp ./test \
  -n helm-demo \
  --set replicaCount=4

helm upgrade myapp ./test \
  -n helm-demo \
  --set replicaCount=3 \
  --set image.tag=latest \
  --set service.type=NodePort

helm history myapp -n helm-demo

helm rollback myapp 1 -n helm-demo

helm uninstall myapp -n helm-demo

helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm search repo bitnami
helm repo list