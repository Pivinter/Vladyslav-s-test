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
1. Простір імен
    - Проєкт структуровано в окремому Namespace (`python-api-ns`) для логічної ізоляції компонентів і спрощення управління доступом через RBAC.

2. Конфігурація
    - ConfigMaps використовуються для зберігання незахищених параметрів (наприклад, назва БД, порт застосунку).
    - Secrets містять чутливі дані — паролі та API-ключі — у форматі Base64.

3. База даних (PostgreSQL)
    - Реалізована за допомогою StatefulSet з PersistentVolumeClaim (5Gi, ReadWriteOnce).
    - Така конфігурація забезпечує стабільну мережеву ідентичність, впорядкований запуск і постійне сховище даних, що є критично важливим для PostgreSQL.

4. API-застосунок
    - Розгортається через Deployment з трьома репліками.
    - Використовується стратегія оновлення RollingUpdate (maxUnavailable: 25%).
    - Оскільки API є stateless, воно легко масштабується горизонтально без простоїв.

5. Сервіси
    - postgres-service — тип ClusterIP (порт 5432) для внутрішнього доступу.
    - app-service — тип LoadBalancer або NodePort (80 → 5000) для зовнішнього доступу до API.

6. Ingress
    - nginx Ingress Controller виступає єдиною точкою входу.
    - Забезпечує SSL/TLS термінацію та маршрутизацію на основі шляхів (path-based routing).

7. Автоматичне масштабування
    - Налаштовано HorizontalPodAutoscaler для API:
    - кількість реплік: від 2 до 10;
    - пороги: CPU — 70%, пам’ять — 80%.

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
1. StatefulSet — найкраще рішення для навчальних проєктів
Для невеликого навчального проєкту StatefulSet надає:
- повний контроль над конфігурацією;
- прозорість у роботі — можна побачити й зрозуміти кожен параметр;
- просте налагодження (debugging) і відстеження поведінки Kubernetes-компонентів.

2. Чому не Helm Chart (Bitnami) для простих випадків
Helm Chart від Bitnami включає понад 50 параметрів і безліч додаткових функцій:
- реплікація бази даних;
- автоматичні бекапи;
- PgBouncer (connection pooling);
- інші enterprise-фічі.
- Для одного PostgreSQL-інстансу в Minikube усе це зайве і лише ускладнює розгортання.

3. Коли варто використовувати Helm
Helm доцільний у production-середовищі, коли потрібні:
- висока доступність (High Availability) — кілька реплік і автоматичний failover;
- кілька середовищ (dev/staging/prod);
- enterprise-функції, як автоматичні резервні копії, моніторинг та масштабування.

Для навчальних і невеликих проєктів — простий StatefulSet залишається більш зрозумілим і гнучким рішенням.

## Environment Configuration: 
### Configuration Hierarchy
```
Secrets (найвищий пріоритет) => Environment Variables => ConfigMaps => Default Values
```
### Security Best Practices
1. Основні правила безпеки
    - Ніколи не комітьте secrets у Git.
    - Додавайте усі файли з секретами до .gitignore.
    - Base64 — це не шифрування.
    - Kubernetes лише кодує значення, але не захищає їх.
    - Для production використовуйте безпечні рішення, такі як:
        - Sealed Secrets;
        - External Secrets Operator;
        - HashiCorp Vault.
    - Налаштуйте RBAC, щоб обмежити доступ до секретів.
    - Регулярно ротуй паролі та ключі доступу.

2. Використання Secrets у проєкті
`postgres-secret.yaml` — містить облікові дані PostgreSQL (POSTGRES_PASSWORD).
`app-secret.yaml` — зберігає DATABASE_URL для API.

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

1. PostgreSQL Service (ClusterIP)
    - Використовується тип ClusterIP для максимальної безпеки — база даних доступна лише всередині кластера.
    - Доступ здійснюється через внутрішній DNS: `postgres-service.python-api-ns.svc.cluster.local:5432.`
    - Такий підхід запобігає зовнішнім атакам і не експонує базу даних у публічну мережу.

2. API Service
    - У Minikube застосовується тип NodePort, оскільки LoadBalancer працює лише через команду minikube tunnel.
    - У хмарних середовищах (GKE, EKS, AKS) використовується LoadBalancer з автоматичним створенням Cloud Load Balancer.
    - Сервіс експонує порт 80, який проксіюється на targetPort 5000 (контейнери з API).

3. Ingress (nginx)
Використовується як єдина точка входу з `hostname api.local`.
Забезпечує централізований контроль над:
- SSL/TLS termination
- Path-based routing (`/api`, `/admin`)
- Host-based routing (піддомени)
- Rate limiting
- Автентифікацією користувачів
Для локального середовища Minikube додається запис у /etc/hosts: `$(minikube ip) api.local`
Це дозволяє реалізувати локальний DNS resolution.

## Scaling:
1. Тип масштабування
Використовується Horizontal Pod Autoscaler (HPA) для автоматичного масштабування API на основі навантаження CPU та пам’яті (memory utilization).

2. Налаштування поведінки
    - Scale up — відбувається швидко (100% за 15 секунд), щоб оперативно реагувати на збільшення навантаження.
    - Scale down — виконується повільніше (50% за 15 секунд), щоб уникнути частого коливання кількості реплік (flapping).

3. Переваги HPA
    - Ефективно працює, коли ресурсні вимоги контейнерів заздалегідь невідомі.
    - Підходить для сезонних або нерівномірних змін навантаження.
    - Забезпечує стабільність кластера завдяки заданим resource requests/limits, що запобігають перевантаженню та resource starvation інших Pod’ів.

## Monitoring and Logging:
1. Моніторинг (Prometheus + Grafana)
    - Використовується Prometheus + Grafana stack, розгорнутий через Helm chart.
    - Це галузевий стандарт (industry standard) для спостереження за станом кластерів Kubernetes.
    - Stack містить понад 50 готових дашбордів і підтримує автоматичне Service Discovery, що дозволяє миттєво підключати нові сервіси без ручного конфігурування.
    - Такий підхід економить тижні розробки порівняно з ручним налаштуванням моніторингу.

2. Логування (Loki Stack)
    - Для централізованого збору логів використовується Loki Stack, інтегрований із Grafana.
    - Логи з API Pods (`stdout` / `stderr`) збираються автоматично — аналогічно до `kubectl logs`.
    - Усі дані доступні для перегляду й аналізу у єдиному Grafana-інтерфейсі, що спрощує налагодження та пошук проблем у системі.

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