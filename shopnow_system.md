# ShopNow — описание системы + HLD (C4 container)

## 1) Идея
**ShopNow** — интернет-магазин: каталог → корзина → оформление заказа → оплата → уведомления.

Ключевая идея: разделить синхронный путь пользователя (web + API) и асинхронную обработку (уведомления, пост-обработка) через очередь сообщений.

## 2) Компоненты системы

### 2.1. Компоненты внутри Kubernetes (namespace `shopnow`)
| Компонент | Роль | Порт (внутри) | Образ (пример) |
|---|---|---:|---|
| `web-frontend` | Web UI / статика | 80 | `nginxdemos/hello:plain-text` |
| `shop-api` | Публичный API (BFF/API-gateway), авторизация, маршрутизация | 80 | `ghcr.io/shopnow/shop-api:1.0.0` *(в yaml — заглушка `traefik/whoami`)* |
| `catalog-service` | Каталог: товары/цены/остатки | 80 | `ghcr.io/shopnow/catalog-service:1.0.0` *(в yaml — `traefik/whoami`)* |
| `order-service` | Заказы: создание/статус, публикация событий в очередь | 80 | `ghcr.io/shopnow/order-service:1.0.0` *(в yaml — `traefik/whoami`)* |
| `notification-worker` | Консюмер очереди: email/SMS/push, ретраи | — | `ghcr.io/shopnow/notification-worker:1.0.0` *(в yaml — `busybox`-заглушка)* |
| `postgres` | База данных (каталог/заказы) | 5432 | `postgres:16-alpine` |
| `rabbitmq` | Очередь сообщений | 5672, 15672 | `rabbitmq:3.13-management` |

Примечание: в манифестах для микросервисов по умолчанию стоят **публичные заглушки**, чтобы можно было поднять кластер локально без сборки. Для “боевого” варианта просто замените `image:` на ваши реальные.

### 2.2. Внешние зависимости (REST/источники данных)
- **Платёжный провайдер** (внешний REST): `pay.example.com:443` (HTTPS). Использует `order-service`.
- **Email/SMS провайдер** (внешний REST): `notify.example.com:443` (HTTPS). Использует `notification-worker`.

### 2.3. Компоненты вне Kubernetes (с портом, IP/DNS, сервером, требованиями)

**Recommendation Engine (Recsys)** — внешний сервис рекомендаций (отдельная VM, не в k8s).
- Назначение: отдаёт рекомендации для `catalog-service`
- DNS: `recsys.shopnow.internal`
- IP: `10.10.20.15`
- Порт: `8443` (HTTPS)
- Сервер: 1 VM (Linux)
- Системные требования:
  - CPU: 4 vCPU
  - RAM: 8 GB
  - Disk: 50 GB SSD
  - Network: ≥ 100 Mbps
- Эксплуатация: systemd unit, автоперезапуск, логирование journald.

В k8s на него заведён `Service type=ExternalName` (имя `recsys-external`), чтобы внутри кластера обращаться по `https://recsys-external:8443`.

## 3) Основные сценарии
1) Пользователь заходит через Ingress на `web-frontend`.
2) Web обращается в `shop-api`.
3) `shop-api` вызывает `catalog-service` и `order-service`.
4) `order-service` пишет в PostgreSQL и публикует событие в RabbitMQ.
5) `notification-worker` читает события и вызывает внешний notify provider.

## 4) HLD диаграмма (C4 container — схема связей)

flowchart LR
  U[Пользователь] -->|HTTPS| I[Ingress (shopnow.local)]

  I --> W[web-frontend]
  I --> A[shop-api]

  A --> C[catalog-service]
  A --> O[order-service]

  C --> P[(PostgreSQL)]
  O --> P

  O --> Q[(RabbitMQ)]
  N[notification-worker] --> Q

  C --> R[recsys-external (VM вне k8s)]
  O --> PAY[Payment provider (внешний REST)]

  N --> NOTIFY[Email/SMS provider (внешний REST)]
