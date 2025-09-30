# 🏗️ Архитектура системы URL Shortener

## Обзор

Это пример микросервисной архитектуры с event-driven подходом, демонстрирующий основные паттерны построения распределённых систем.

## Диаграмма архитектуры

```
┌─────────────┐
│   Browser   │
│  (Client)   │
└──────┬──────┘
       │
       │ HTTP
       ▼
┌─────────────────────────────────────────────────────────────┐
│                        Frontend                              │
│                     (Nginx + HTML)                           │
│                      Port: 8080                              │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       │ REST API
                       ▼
       ┌───────────────────────────────┐
       │       API Gateway             │
       │         (Go)                  │
       │       Port: 3000              │
       └───────┬──────────────┬────────┘
               │              │
               │              │
      ┌────────▼────┐    ┌───▼────────┐
      │ Shortener   │    │ Analytics  │
      │  Service    │    │  Service   │
      │   (Go)      │    │   (Go)     │
      │ Port: 3001  │    │ Port: 3003 │
      └──────┬──────┘    └─────┬──────┘
             │                 │
             │                 │ Read
             ▼                 ▼
      ┌──────────┐      ┌─────────────┐
      │  Redis   │      │  MongoDB    │
      │  Cache   │      │  Database   │
      │Port: 6379│      │ Port: 27017 │
      └─────▲────┘      └──────▲──────┘
            │                  │
            │                  │ Write
            │           ┌──────┴──────┐
            │           │   Kafka     │
            │           │  Consumer   │
            │           │  (part of   │
            │           │  analytics) │
            │           └──────▲──────┘
            │                  │
            │                  │ Events
            │           ┌──────┴──────┐
            │           │   Kafka     │
            │           │   Broker    │
            │           │ Port: 9092  │
            │           └──────▲──────┘
            │                  │
            │                  │ Publish
            │           ┌──────┴──────┐
      ┌─────▼─────┐     │   Kafka     │
      │ Redirect  │─────┤  Producer   │
      │  Service  │     │  (part of   │
      │   (Go)    │     │  redirect)  │
      │Port: 3002 │     └─────────────┘
      └───────────┘

      ┌──────────────┐
      │  Zookeeper   │
      │ (for Kafka)  │
      │ Port: 2181   │
      └──────────────┘
```

## Компоненты системы

### 1. Frontend (Nginx + HTML)
**Назначение:** Веб-интерфейс для пользователей

**Технологии:**
- HTML5
- CSS3
- Vanilla JavaScript
- Nginx

**Функции:**
- Создание коротких URL
- Просмотр статистики
- Копирование ссылок

---

### 2. API Gateway
**Назначение:** Единая точка входа для всех API запросов

**Технологии:**
- Go 1.21
- Gorilla Mux (роутинг)
- CORS middleware

**Паттерны:**
- API Gateway
- Reverse Proxy

**Функции:**
- Маршрутизация запросов к соответствующим сервисам
- Централизованная обработка CORS
- Логирование всех запросов
- Health checks для downstream сервисов

**Endpoints:**
```
POST   /api/shorten       → shortener-service
GET    /api/stats/{code}  → analytics-service
GET    /api/stats         → analytics-service
GET    /api/info          → system info
GET    /health            → health check
```

---

### 3. Shortener Service
**Назначение:** Создание коротких URL

**Технологии:**
- Go 1.21
- Redis client (go-redis/redis/v8)

**Хранилище:** Redis

**Алгоритм:**
1. Получить URL от клиента
2. Сгенерировать уникальный 6-символьный код (base62)
3. Проверить уникальность в Redis
4. Сохранить маппинг `url:{code} → original_url`
5. Вернуть короткий URL

**Генерация кода:**
- Charset: `a-zA-Z0-9` (62 символа)
- Длина: 6 символов
- Возможных комбинаций: 62^6 ≈ 56 миллиардов

**Endpoints:**
```
POST /shorten    - создать короткий URL
GET  /health     - health check
```

---

### 4. Redirect Service
**Назначение:** Перенаправление по коротким URL

**Технологии:**
- Go 1.21
- Redis client
- Kafka Producer (segmentio/kafka-go)

**Хранилище:** Redis (чтение)

**Поток:**
1. Получить короткий код из URL
2. Найти оригинальный URL в Redis
3. **Асинхронно** отправить событие клика в Kafka
4. HTTP 302 редирект на оригинальный URL

**Kafka событие:**
```json
{
  "shortCode": "abc123",
  "timestamp": "2024-01-01T12:00:00Z",
  "userAgent": "Mozilla/5.0...",
  "ip": "192.168.1.1"
}
```

**Endpoints:**
```
GET /{shortCode}  - перенаправление
GET /health       - health check
```

**Особенности:**
- Асинхронная отправка в Kafka (не блокирует редирект)
- Быстрый ответ пользователю
- Fault tolerant (если Kafka недоступна, редирект всё равно работает)

---

### 5. Analytics Service
**Назначение:** Сбор и предоставление статистики

**Технологии:**
- Go 1.21
- MongoDB driver
- Kafka Consumer (segmentio/kafka-go)

**Хранилище:** MongoDB

**Компоненты:**
1. **Kafka Consumer (горутина)**
   - Читает события из топика `url-clicks`
   - Сохраняет в MongoDB коллекцию `clicks`
   - Consumer Group: `analytics-consumer-group`

2. **HTTP API (основной поток)**
   - Предоставляет статистику по запросу
   - Агрегирует данные из MongoDB

**Endpoints:**
```
GET /stats/{shortCode}  - статистика для конкретного URL
GET /stats              - вся статистика (агрегированная)
GET /health             - health check
```

**MongoDB схема:**
```javascript
{
  shortCode: String,     // индексированное поле
  timestamp: ISODate,
  userAgent: String,
  ip: String
}
```

**Агрегация:**
- Группировка по `shortCode`
- Подсчёт количества (`$sum`)
- Последний клик (`$max timestamp`)

---

### 6. Redis
**Назначение:** In-memory хранилище для URL маппинга

**Версия:** 7 Alpine

**Структура данных:**
```
Key: url:{shortCode}
Value: {originalUrl}
TTL: бесконечно (можно добавить)
```

**Почему Redis:**
- Очень быстрое чтение (< 1ms)
- Простая key-value модель
- Persistence (RDB/AOF)
- Репликация для HA

---

### 7. MongoDB
**Назначение:** Хранилище событий и статистики

**Версия:** 7

**База данных:** `analytics`
**Коллекция:** `clicks`

**Индексы:**
- `shortCode` (для быстрых запросов)

**Почему MongoDB:**
- Гибкая схема (можно добавлять поля)
- Мощные агрегации
- Хорошо для event store
- Горизонтальное масштабирование (sharding)

---

### 8. Kafka
**Назначение:** Message broker для асинхронного взаимодействия

**Версия:** Confluent Platform 7.5.0

**Топики:**
- `url-clicks` - события переходов

**Consumer Groups:**
- `analytics-consumer-group` - analytics-service

**Конфигурация:**
- Replication factor: 1 (для demo)
- Auto-create topics: enabled
- Retention: по умолчанию

**Почему Kafka:**
- High throughput
- Durability (persistence на диск)
- Масштабируемость
- Decoupling (producer/consumer независимы)
- Replay возможность

---

### 9. Zookeeper
**Назначение:** Координация Kafka кластера

**Версия:** Confluent 7.5.0

**Функции:**
- Управление метаданными Kafka
- Leader election для партиций
- Consumer group coordination

---

## Потоки данных

### Сценарий 1: Создание короткого URL

```
User → Frontend → API Gateway → Shortener Service → Redis
                                                    ↓
User ← Frontend ← API Gateway ← Shortener Service ←┘
```

**Шаги:**
1. Пользователь вводит URL в форму
2. Frontend отправляет POST /api/shorten
3. API Gateway проксирует к shortener-service
4. Shortener генерирует код и сохраняет в Redis
5. Возвращает короткий URL пользователю

**Время отклика:** ~10-50ms

---

### Сценарий 2: Переход по короткому URL

```
User → Redirect Service → Redis (get URL)
  ↑                    ↓
  └─── HTTP 302 ───────┘
                       ↓
                    Kafka (async publish)
                       ↓
                 Analytics Service
                       ↓
                    MongoDB
```

**Шаги:**
1. Пользователь открывает короткую ссылку
2. Redirect Service получает код
3. Читает оригинальный URL из Redis
4. **Асинхронно** отправляет событие в Kafka
5. Делает HTTP 302 редирект
6. Analytics Service читает из Kafka
7. Сохраняет в MongoDB

**Время отклика:** ~5-20ms (не ждёт Kafka/MongoDB)

---

### Сценарий 3: Просмотр статистики

```
User → Frontend → API Gateway → Analytics Service → MongoDB
                                                    ↓
User ← Frontend ← API Gateway ← Analytics Service ←┘
```

**Шаги:**
1. Пользователь нажимает "Обновить статистику"
2. Frontend запрашивает GET /api/stats
3. API Gateway проксирует к analytics-service
4. Analytics выполняет агрегацию в MongoDB
5. Возвращает результаты

**Время отклика:** ~50-200ms (зависит от объёма данных)

---

## Паттерны микросервисов

### 1. API Gateway Pattern
Единая точка входа скрывает сложность внутренней архитектуры.

**Преимущества:**
- Упрощение клиента
- Централизованная аутентификация
- Rate limiting в одном месте
- SSL termination

### 2. Database per Service
Каждый сервис имеет свою БД.

**Преимущества:**
- Независимое развитие
- Технологическое разнообразие
- Изолированные сбои

**Недостатки:**
- Нет ACID транзакций между сервисами
- Сложность консистентности

### 3. Event-Driven Architecture
Асинхронное взаимодействие через события.

**Преимущества:**
- Loose coupling
- Scalability
- Resilience
- Temporal decoupling

**Недостатки:**
- Eventual consistency
- Сложность отладки
- Ordering challenges

### 4. CQRS (Command Query Responsibility Segregation)
Разделение записи (redirect-service) и чтения (analytics-service).

**Преимущества:**
- Оптимизация под разные нагрузки
- Независимое масштабирование
- Разные модели данных

---

## Масштабирование

### Горизонтальное масштабирование

**Shortener Service:**
```bash
docker-compose up -d --scale shortener-service=3
```
- Load balancing через Docker DNS round-robin
- Stateless сервис

**Redirect Service:**
```bash
docker-compose up -d --scale redirect-service=5
```
- Можно ставить за Nginx/HAProxy
- Kafka producer в каждом инстансе

**Analytics Service:**
```bash
docker-compose up -d --scale analytics-service=3
```
- Kafka автоматически распределит партиции между consumer'ами
- Consumer group обеспечивает балансировку

---

## Отказоустойчивость

### Точки отказа и решения

1. **Redis упал**
   - Shortener: не может создавать URL ❌
   - Redirect: не может перенаправлять ❌
   - Решение: Redis Sentinel / Redis Cluster

2. **MongoDB упала**
   - Analytics: не может сохранять статистику ❌
   - Analytics API: не может отдавать статистику ❌
   - Redirect: продолжает работать ✅
   - Решение: MongoDB Replica Set

3. **Kafka упала**
   - Redirect: продолжает перенаправлять ✅ (асинхронная отправка)
   - Analytics: не получает новые события ❌
   - Решение: Kafka Cluster (3+ brokers)

4. **Analytics Service упала**
   - Redirect: продолжает работать ✅
   - События накапливаются в Kafka ✅
   - При восстановлении обработает всё ✅

---

## Мониторинг и Observability

### Метрики (будущее улучшение)
- Prometheus для сбора метрик
- Grafana для визуализации

### Логирование
- Структурированные логи
- Корреляция через request-id

### Трейсинг (будущее)
- OpenTelemetry
- Jaeger/Zipkin

---

## Безопасность

### Текущее состояние (demo)
- Нет аутентификации
- Нет rate limiting
- Нет валидации URL

### Рекомендации для production
- JWT токены
- Rate limiting (Redis)
- URL validation & sanitization
- HTTPS everywhere
- Network policies
- Secret management (Vault)

---

## Тестирование

### Unit тесты
- Тестирование бизнес-логики каждого сервиса
- Моки для внешних зависимостей

### Integration тесты
- Тестирование взаимодействия с БД
- Testcontainers для Redis/MongoDB/Kafka

### E2E тесты
- Полный сценарий через API
- Docker Compose окружение

---

## Deployment

### Текущий способ: Docker Compose
Подходит для:
- Локальная разработка
- Небольшие deployment'ы
- Demo/POC

### Kubernetes (рекомендуется для production)
```
- Deployments для каждого сервиса
- Services для service discovery
- ConfigMaps для конфигурации
- Secrets для паролей
- StatefulSets для Kafka/ZooKeeper
- Persistent Volumes для данных
- Horizontal Pod Autoscaler
- Ingress для внешнего доступа
```

---

## Выводы

Этот проект демонстрирует:

✅ Разделение ответственности между сервисами  
✅ Асинхронное взаимодействие через message broker  
✅ Event sourcing для аналитики  
✅ Масштабируемость каждого компонента  
✅ Fault tolerance через decoupling  
✅ Технологическое разнообразие (полиглот персистенция)  

Это foundation для изучения более сложных концепций:
- Service Mesh (Istio, Linkerd)
- API Gateway продвинутые фичи (Kong, Ambassador)
- Distributed Tracing
- Circuit Breaker pattern
- Saga pattern для распределённых транзакций
