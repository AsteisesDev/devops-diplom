# Дипломный проект: Развертывание и мониторинг облачного веб-приложения с CI/CD

Дипломная работа по специальности DevOps-инженер. Проект демонстрирует развёртывание отказоустойчивого веб-приложения в Kubernetes с полным циклом доставки, наблюдаемостью и безопасным доступом.

## Стек технологий

- **Kubernetes** (Minikube) — оркестрация контейнеров
- **Helm** — пакетный менеджер для Kubernetes
- **Docker** — контейнеризация приложения
- **Nginx** — веб-сервер
- **GitHub Actions** — автоматизация сборки и развёртывания
- **Prometheus + Grafana** — мониторинг и визуализация метрик
- **Loki + Promtail** — сбор и просмотр логов

## Пререквизиты

- Docker
- Minikube
- kubectl
- Helm 3
- GitHub-аккаунт

## Быстрый старт

Все команды выполняются в **WSL (Ubuntu)**.

```bash
cd /mnt/d/projects/diplom/devops-diplom
```

### 1. Запуск кластера

```bash
bash scripts/setup-minikube.sh
```

Скрипт автоматически:
- Запускает Minikube с Docker-драйвером
- Загружает образы ingress-nginx в Minikube (обход блокировки registry.k8s.io)
- Устанавливает ingress-nginx через Helm (minikube addon не используется из-за сетевых ограничений)
- Включает metrics-server

### 2. Развёртывание приложения

```bash
helm install webapp helm/webapp
```

Проверка статуса:

```bash
kubectl get pods
kubectl get svc
kubectl get ingress
```

### 3. Доступ к приложению

Через port-forward (самый простой способ):

```bash
kubectl port-forward svc/webapp 9090:80
```

Откройте: http://localhost:9090

Через Ingress (требует дополнительной настройки):

```bash
# В отдельном терминале:
minikube tunnel
```

Добавьте в `/etc/hosts`:

```
127.0.0.1 webapp.local
```

Откройте: http://webapp.local

### 4. Установка мониторинга

```bash
bash monitoring/install.sh
```

Скрипт автоматически:
- Загружает все необходимые Docker-образы в Minikube
- Устанавливает Loki + Promtail для сбора логов
- Устанавливает kube-prometheus-stack (Prometheus + Grafana 10.4.15)
- Настраивает Loki как datasource в Grafana

> **Важно:** используется Grafana 10.4.15, а не последняя версия.
> Grafana 12.x несовместима со встроенным Loki-плагином.

Доступ к Grafana:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

Откройте http://localhost:3000 (логин: `admin`, пароль: `admin`).

**Метрики:** Dashboards → встроенные дашборды Kubernetes (Compute Resources / Pod и др.)

**Логи:** Explore → выберите datasource **Loki** → запрос `{namespace="default"}`

> Ошибка "Failed to load log volume" в Explore не влияет на отображение логов.
> Это известная проблема совместимости Grafana 10 с Loki 2.6.

### 5. CI/CD

Проект использует GitHub Actions с тремя задачами:

1. **build** — сборка Docker-образа и push в GitHub Container Registry (ghcr.io)
2. **test** — линтинг и валидация Helm-чарта
3. **deploy** — развёртывание в кластер (через self-hosted runner)

Pipeline запускается автоматически при push в `main`.

## Структура проекта

```
├── app/                          # Веб-приложение
│   ├── Dockerfile
│   └── html/index.html
├── helm/webapp/                  # Helm-чарт
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/                # Шаблоны K8s-ресурсов
├── monitoring/
│   └── install.sh                # Установка Prometheus/Grafana/Loki
├── scripts/
│   └── setup-minikube.sh         # Инициализация кластера
├── docs/
│   └── report.md                 # Дипломный отчёт
└── .github/workflows/ci.yml     # CI/CD pipeline
```

## Проверка RBAC

```bash
# Должно вернуть "yes"
kubectl auth can-i list pods --as=system:serviceaccount:default:webapp-sa

# Должно вернуть "no"
kubectl auth can-i delete deployments --as=system:serviceaccount:default:webapp-sa
```
