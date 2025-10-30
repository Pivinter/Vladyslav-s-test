Вітаю того хто буде приймати моє завдання. 
# Application: Python REST API with PostgreSQL
API підтримує базові CRUD операції:
- `GET /health` - перевірка здоров'я додатку.
- `GET /api/items` - отримати всі items.
- `GET /api/items/<id>` - отримати конкретний item.
- `POST /api/items` - створити новий item.
- `PUT /api/items/<id>` - оновити item.
- `DELETE /api/items/<id>` - видалити item.

# Deployment Target: 
Kubernetes cluster: Minikube.

# Plan Contents:
## Infrastructure:
    Структурую проєкт у окремому Namespace (python-api-ns) для логічної ізоляції та легшого RBAC управління. 
Використовую ConfigMaps для незахищеної конфігурації (назва БД, порт додатку) 
та Secrets для чутливих даних (паролі, API ключі) у base64 encoding.
Для БД використовую StatefulSet з PersistentVolumeClaim (5Gi, ReadWriteOnce), 
о скільки PostgreSQL потребує стабільної мережевої ідентичності, впорядкованого запуску 
та постійного сховища. 
API розгортаю через Deployment з трьома репліками та RollingUpdate стратегією 
(25% maxUnavailable), бо API є stateless і легко масштабується горизонтально без downtime.
Створюю два Services: postgres-service (ClusterIP на порту 5432) для внутрішнього доступу 
та app-service (LoadBalancer/NodePort, 80 => 5000) для зовнішнього. 
Додаю Ingress (nginx) як єдину точку входу з SSL/TLS термінацією та path-based routing. 
Для автомасштабування налаштовую HorizontalPodAutoscaler (2-10 реплік, CPU 70%, Memory 80%).

## Containerization:
Вибір базового образу:
```dockerfile
FROM python:3.11-slim
```
python:3.11-slim - оптимальний вибір оскільки розмір: ~150MB, містить всі необхідні системні 
бібліотеки та регулярні security updates

Альтернативи :
`python:3.11-alpine` - занадто мінімальний, проблеми з компіляцією psycopg2
`python:3.11` - надто великий (1GB+)

### Docker Image Registry
Для Minikube:
```bash
eval $(minikube docker-env)
docker build -t python-api:v1.0 .
```
Для Production:
Docker Hub: `username/python-api:v1.0`
Azure Container Registry, AWS ECR, GCP GCR

## Database:
### Вибрана Стратегія: StatefulSet
    Для невеликого навчального проєкту StatefulSet дає повний контроль і прозорість можна бачити
кожен параметр, розумієш як працює Kubernetes, і легко дебажити проблеми. Helm chart від 
Bitnami - це 50+ параметрів і купа зайвих features (реплікація, бекапи, PgBouncer), які не
потрібні для простого одного PostgreSQL instance в Minikube.
    Коли використовувати Helm: тільки в production, коли потрібна High Availability 
(кілька реплік з автоматичним failover), багато різних environments (dev/staging/prod), 
або enterprise features типу автоматичних бекапів і моніторингу. Для навчання та простих
проєктів - StatefulSet простіший і зрозуміліший.

## Environment Configuration: 
### Configuration Hierarchy
```
Secrets (найвищий пріоритет) => Environment Variables => ConfigMaps => Default Values
```
### Security Best Practices
Ніколи не комітити secrets в Git і обов'язково додавати secret files до .gitignore. 
Base64 encoding в Kubernetes - це НЕ шифрування, тому для production потрібно 
використовувати спеціалізовані рішення типу Sealed Secrets від Bitnami,  External Secrets 
Operator або HashiCorp Vault.  Також важливо налаштувати RBAC для обмеження доступу до 
secrets і регулярно ротувати паролі.
Secrets: PostgreSQL credentials зберігаю в postgres-secret.yaml (POSTGRES_PASSWORD)
DATABASE_URL для API в app-secret.yaml
Кодування: Base64 (стандарт K8s)

### Environment-specific Configs
Для різних середовищ:
```
k8s/
base/ # Спільна конфігурація
overlays/
    dev/ # Development
        staging/ # Staging
        prod/ # Production (різні secrets, replicas)
```

## Networking:
### Network Architecture

```
Internet/Browser => [Ingress] => [app-service:80] (LoadBalancer/NodePort) => [API Pods:5000]
(3 replicas) => [postgres-service:5432] (ClusterIP) => [PostgreSQL Pod:5432]
```

    PostgreSQL Service (ClusterIP): Використовую ClusterIP для максимальної безпеки - БД доступна
тільки всередині кластера через внутрішній DNS (postgres-service.python-api-ns.svc.cluster.
local:5432), що запобігає зовнішнім атакам.

    API Service: Для Minikube використовую NodePort (LoadBalancer працює через minikube tunnel),
для Cloud провайдерів (GKE/EKS/AKS) - LoadBalancer з автоматичним Cloud LB. Service експонує
порт 80 який проксує на targetPort 5000 API pods.

    Ingress (nginx): Налаштовую як єдину точку входу з hostname api.local, що дає централізований 
контроль над SSL/TLS termination, path-based routing (/api, /admin), host-based routing 
(піддомени), rate limiting та автентифікацією. Для Minikube додаю $(minikube ip) api.local в 
/etc/hosts для локального DNS resolution.

## Scaling:
Scaling Strategy: Використовую Horizontal Pod Autoscaler (HPA) для автоматичного масштабування
API на основі CPU/memory utilization - scale up відбувається швидко (100% за 15 секунд) 
для негайної реакції на навантаження, а scale down повільно (50% за 15 секунд) 
щоб уникнути flapping. HPA особливо корисний для невідомих resource requirements
та сезонних змін у навантаженні, а також встановлюю resource limits (requests/limits) 
для забезпечення стабільності та запобігання resource starvation інших pods.

## Monitoring and Logging:
Використовую Prometheus + Grafana stack через Helm chart, оскільки це industry standard з 50+
готовими dashboards та автоматичним Service discovery, що економить тижні розробки порівняно з
ручним налаштуванням.  Для логування використовую Loki Stack інтегрований з Grafana, де логи з
API pods (stdout/stderr) автоматично збираються через kubectl logs і доступні для аналізу в 
централізованому інтерфейсі.

#### Встановлення:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace
```
Моніторю критичні метрики (CPU/Memory, request rate, response time, error rate) та важливі
показники (database connections, disk usage, pod restarts). 

### Logging Strategy
Architecture:
```
API Pods => stdout/stderr => kubectl logs => Loki => Grafana
```
Встановлення Loki Stack:
```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --set grafana.enabled=true \
  --set prometheus.enabled=true \
  --namespace monitoring
```