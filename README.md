# Momo Store aka Пельменная №2

<img width="900" alt="image" src="https://user-images.githubusercontent.com/9394918/167876466-2c530828-d658-4efe-9064-825626cc6db5.png">

## Frontend

```bash
npm install
NODE_ENV=production VUE_APP_API_URL=http://localhost:8081 npm run serve
```

## Backend

```bash
go run ./cmd/api
go test -v ./... 
```

UPD: Про домен и метрики. Я уже хочу быстрее это все закончить, поэтому я особо не заморачивался с днс, да и не знаю где его регистрировать оплачивать и все подобное. Я развернул графану и прометей, но на сайте нет кнопки чтобы завершить заказ и поэтому данных о заказах нет.

## 🚀 Продуктивная среда  

- **Основное приложение**: https://158.160.141.146.nip.io/  
- **Метрики (Prometheus)**: https://prom.158.160.141.146.nip.io
- **Мониторинг (Grafana)**: https://grafana.158.160.141.146.nip.io/

## 📁 Структура проекта  

momo-store/  
├── backend/ - Go API сервер  
│ ├── Dockerfile - Образ Docker  
│ └── .gitlab-ci.yml - CI/CD для бэкенда  
├── frontend/ - Vue.js приложение  
│ ├── Dockerfile - Образ Docker  
│ └── .gitlab-ci.yml - CI/CD для фронтенда  
├── infrastructure/ - Инфраструктура как код  
│ ├── terraform/ - Terraform конфигурации  
│ │ ├── main.tf - Основные ресурсы  
│ │ ├── variables.tf - Переменные  
│ │ ├── provider.tf - Провайдеры  
│ │ └── versions.tf - Версии провайдеров  
│ ├── momo-store-chart/ - Главный Helm chart  
│ │ ├── charts/ - Subcharts  
│ │ │ ├── backend/ - Chart для бэкенда  
│ │ │ ├── frontend/ - Chart для фронтенда  
│ │ │ ├── grafana/ - Chart для Grafana  
│ │ │ ├── prometheus/ - Chart для Prometheus  
│ │ │ └── ingress/ - Chart для Ingress  
│ │ ├── Chart.yaml - Метаданные chart  
│ │ └── values.yaml - Значения по умолчанию  
│ ├── service-account/ - Сервисные аккаунты K8s  
│ └── .gitlab-ci.yml - CI/CD для инфраструктуры  
├── .gitlab-ci.yml - Главный CI/CD пайплайн  
└── README.md - Документация  

## 🏗️ Развертывание инфраструктуры  

### Предварительные требования  

1. **Аккаунт Yandex Cloud** с доступом к:  
   - Yandex Managed Kubernetes  
   - Yandex Object Storage  
   - Service accounts  

2. **Установленное ПО**:  
```bash
   terraform >= 1.3.0
   kubectl >= 1.25.0
   helm >= 3.8.0
   yc CLI
```

### Настройка Terraform  

1. **Создать сервисный аккаунт для Terraform**:  
```bash
yc iam service-account create --name sa-terraform
yc resource-manager folder add-access-binding \
  --role editor \
  --subject serviceAccount:<sa-terraform-id>
```
2. **Создать статический ключ доступа**:  
```bash
yc iam access-key create --service-account-name sa-terraform
```
3. **Настроить файлы конфигурации**:  
infrastructure/terraform/backend.tfvars:  
```bash
access_key = "your_access_key_here"
secret_key = "your_secret_key_here"
bucket    = "tf-state-momo-store"
```  
infrastructure/terraform/secret.tfvars:  
```bash
token = "$(yc iam create-token)"
```  

### Создание инфраструктуры  

cd infrastructure/terraform  

 **Инициализация Terraform**  
```bash
terraform init -backend-config=backend.tfvars
```
 **Планирование развертывания**
```bash
terraform plan -var-file="secret.tfvars"
```
 **Применение конфигурации**
```bash
terraform apply -var-file="secret.tfvars"
```

## 🔄 Правила внесения изменений в инфраструктуру

### Процесс изменений

1. **Создание feature ветки от main:**  
```bash
git checkout -b feature/terraform-<change-description>
```  
2. **Внесение изменений в Terraform конфигурации**  
3. **Проверка плана изменений:**  

```bash 
terraform plan -var-file="secret.tfvars"
```

## 🚀 Развертывание приложения

### Подготовка Kubernetes  

1. **Настройка kubectl:**  
```bash
yc managed-kubernetes cluster get-credentials \
  --id <cluster-id> \
  --external
```  
2. **Создание namespace:**  
```bash
kubectl create namespace momo-store
```
3. **Установка cert-manager:**  
```bash
helm repo add jetstack https://charts.jetstack.io  
helm repo update  
helm upgrade --install --atomic \  
  -n momo-store \  
  cert-manager jetstack/cert-manager \  
  --set installCRDs=true  
``` 

### Настройка GitLab CI/CD  

 **Добавить переменные в GitLab (Settings → CI/CD → Variables):**
Переменная	Значение
KUBE_CONFIG	base64(kubeconfig)  
NEXUS_USER	nexus username  
NEXUS_PASS	nexus password  
NEXUS_HELM_REPO	nexus repo url  
GRAFANA_ADM_PWD	admin password  

### Установка приложения  

```bash
# Добавление Helm репозитория  
helm repo add momo-store \
  http://nexus.praktikum-services.tech/repository/std-000-00-momo-store-helm/
helm repo update

# Установка приложения
helm upgrade --install --atomic \
  -n momo-store \
  momo-store ./infrastructure/momo-store-chart \
  --set backend.image.tag=1.0.0 \
  --set frontend.image.tag=1.0.0
  --set grafana.adminPassword=momostore
```

### Проверка установки  

```bash
# Статус подов
kubectl get pods -n momo-store

# Статус сервисов
kubectl get svc -n momo-store

# Логи приложения
kubectl logs -n momo-store -l app=frontend
```

## 🔄 Релизный цикл и версионирование  

### Правила версионирования (SemVer)  

- Формат: MAJOR.MINOR.PATCH  
- MAJOR: Критические изменения, ломающие обратную совместимость  
- MINOR: Новая функциональность с сохранением совместимости  
- PATCH: Исправления багов и мелкие улучшения  

### Установка версий  

В файле .gitlab-ci.yml:  
```bash
variables:  
  VERSION: "1.0.0"  # MAJOR.MINOR устанавливаются вручную
  # PATCH автоматически устанавливается из CI_PIPELINE_ID
```

### Процесс релиза  

1. Разработка в feature ветках  
2. Создание Merge Request в main:  
	- Прохождение code review  
	- Успешное выполнение CI/CD пайплайна  
	- Обновление версии при необходимости  
3. Автоматический деплой при мерже в main:  
	- Сборка Docker образов с тегом {VERSION}.{CI_PIPELINE_ID}  
	- Публикация в GitLab Container Registry  
	- Развертывание в Kubernetes кластере  
	- Публикация Helm chart в Nexus  

### CI/CD Pipeline этапы  

 **pre-build**:  
	- SAST анализ (Semgrep)  
	- Поиск секретов (Gitleaks)  
	- SCA анализ (Trivy)  
 **build**:  
	- Сборка Docker образов  
	- Тестирование образов на уязвимости  
 **test**:  
	- Unit тесты бэкенда  
	- Интеграционные тесты  
 **deploy**:  
	- Деплой в Kubernetes  
	- Публикация Helm chart  

## 📊 Мониторинг отсутсвует, по выше указанным причинам 

 Полезные ссылки  
Gitlab: https://gitlab.praktikum-services.ru/std-043-28/momo-store.git  
Production: https://158.160.141.146.nip.io  
Helm Registry: http://nexus.praktikum-services.tech/repository/momo-std-043-28/
