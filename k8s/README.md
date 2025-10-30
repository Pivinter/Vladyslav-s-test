# Kubernetes Configuration Files
Ця директорія містить всі Kubernetes manifest файли для деплойменту Python REST API з PostgreSQL.

## Deployment Instructions
### 1. Prerequisites
```bash
minikube start --cpus=4 --memory=8192 --driver=docker

minikube addons enable ingress
minikube addons enable metrics-server
```

### 2. Build Docker Image
```bash
eval $(minikube docker-env)

docker build -t python-api:v1.0 .
```

### 3. Deploy to Kubernetes
```bash
kubectl apply -f namespace.yaml
kubectl apply -f postgres/
kubectl apply -f app/
kubectl apply -f ingress/
kubectl apply -f autoscaling/
```

### 4. Verify Deployment
```bash
kubectl get all -n python-api-ns
kubectl get pods -n python-api-ns -w
kubectl get svc -n python-api-ns
kubectl get ingress -n python-api-ns
```

### 5. Access Application
#### Option A: Через NodePort
```bash
minikube ip
# Доступ до API
curl http://$(minikube ip):30080/health
curl http://$(minikube ip):30080/api/items
```

#### Option B: Через Ingress
```bash
# Додати в /etc/hosts
echo "$(minikube ip) api.local" | sudo tee -a /etc/hosts

# Доступ до API
curl http://api.local/health
curl http://api.local/api/items
```

## Testing
### Health Check
```bash
curl http://api.local/health
```

### Create Item
```bash
curl -X POST http://api.local/api/items \
  -H "Content-Type: application/json" \
  -d '{"name": "Test Item", "description": "From Kubernetes"}'
```

### Get All Items
```bash
curl http://api.local/api/items
```

### Get Specific Item
```bash
curl http://api.local/api/items/1
```

### Update Item
```bash
curl -X PUT http://api.local/api/items/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Updated Item", "description": "Updated in K8s"}'
```

### Delete Item
```bash
curl -X DELETE http://api.local/api/items/1
```

## Monitoring

### View Logs
```bash
kubectl logs -f deployment/python-api -n python-api-ns
kubectl logs -f statefulset/postgres -n python-api-ns
kubectl logs -f -l app=python-api -n python-api-ns
```

### Check HPA
```bash
kubectl get hpa -n python-api-ns
kubectl get hpa -n python-api-ns -w
```

### Resource Usage
```bash
# CPU/Memory usage
kubectl top pods -n python-api-ns
# Node usage
kubectl top nodes
```

## Update Deployment
### Update Docker Image
```bash
docker build -t python-api:v1.1 .
kubectl set image deployment/python-api api=python-api:v1.1 -n python-api-ns
kubectl rollout status deployment/python-api -n python-api-ns
```

### Rollback
```bash
# Rollback до попередньої версії
kubectl rollout undo deployment/python-api -n python-api-ns
# Rollback до конкретної ревізії
kubectl rollout undo deployment/python-api --to-revision=2 -n python-api-ns
# Історія rollouts
kubectl rollout history deployment/python-api -n python-api-ns
```

## Cleanup
### Delete Everything
```bash
kubectl delete namespace python-api-ns
```
### Delete Docker Image
```bash
docker rmi python-api:v1.0
```