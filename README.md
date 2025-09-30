# 🔗 URL Shortener - Демонстрация Микросервисной Архитектуры

Пример микросервисной архитектуры на **Go** с использованием **Kafka**, **Redis** и **MongoDB**.

> 📖 **Новичок?** Начните с [QUICKSTART.md](./QUICKSTART.md) - за 3 минуты до работающей системы!

## 🏗️ Архитектура

Проект состоит из следующих микросервисов:

### Микросервисы

1. **api-gateway** (Go)
   - Точка входа для всех API запросов
   - Маршрутизация к соответствующим сервисам
   - Порт: `3000`

2. **shortener-service** (Go + Redis)
   - Создание коротких URL
   - Хранение маппинга в Redis
   - Генерация уникальных кодов
   - Порт: `3001`

3. **redirect-service** (Go + Redis + Kafka Producer)
   - Перенаправление по коротким URL
   - Получение оригинального URL из Redis
   - Отправка событий кликов в Kafka
   - Порт: `3002`

4. **analytics-service** (Go + MongoDB + Kafka Consumer)
   - Потребление событий из Kafka
   - Сохранение статистики в MongoDB
   - Предоставление API для статистики
   - Порт: `3003`

5. **frontend** (HTML + Nginx)
   - Веб-интерфейс для демонстрации
   - Порт: `8080`

### Инфраструктура

- **Redis** - хранилище URL маппинга (in-memory)
- **MongoDB** - хранилище статистики переходов
- **Kafka** + **Zookeeper** - шина сообщений для асинхронного взаимодействия между сервисами
- **Jaeger** - distributed tracing для мониторинга и отладки запросов

## 🎯 Демонстрируемые концепции

✅ **Микросервисная архитектура** - каждый сервис выполняет свою функцию  
✅ **Event-Driven Architecture** - асинхронное взаимодействие через Kafka  
✅ **Разделение ответственности** - каждый сервис имеет свою БД  
✅ **API Gateway паттерн** - единая точка входа  
✅ **Service Discovery** - сервисы находят друг друга через Docker DNS  
✅ **Масштабируемость** - сервисы можно масштабировать независимо  
✅ **Fault Tolerance** - падение одного сервиса не роняет всю систему  
✅ **Distributed Tracing** - Jaeger + OpenTelemetry для отслеживания запросов  
✅ **Shared Libraries** - переиспользуемые компоненты (`pkg/tracing`)  
✅ **CI/CD** - GitHub Actions с динамической матрицей для сборки образов  
✅ **Multi-Platform** - поддержка AMD64 и ARM64 (Apple Silicon, Intel, ARM серверы)  

## 🚀 Быстрый старт

### Требования

- Docker
- Docker Compose
- Make (опционально, для удобства)

### Запуск

#### Вариант 1: С использованием Make (рекомендуется)

```bash
# Посмотреть все доступные команды
make help

# Собрать и запустить все сервисы
make up-build

# Установить git hooks (автодобавление ID issue)
make hook

# Проверить здоровье всех сервисов
make health

# Запустить тестовый сценарий
make test

# Посмотреть логи
make logs
```

#### Вариант 2: Напрямую через Docker Compose

```bash
# Запустить все сервисы
docker-compose up --build

# Или в фоновом режиме
docker-compose up -d --build
```

### Проверка работы

1. Откройте браузер: http://localhost:8080
2. Введите любой URL для сокращения
3. Используйте сокращённую ссылку
4. Посмотрите статистику переходов
5. **🔍 Откройте Jaeger UI:** http://localhost:16686 - отследите путь запроса через все сервисы!

### API Endpoints

#### API Gateway (http://localhost:3000)

```bash
# Создать короткий URL
curl -X POST http://localhost:3000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'

# Получить статистику для конкретного кода
curl http://localhost:3000/api/stats/{shortCode}

# Получить всю статистику
curl http://localhost:3000/api/stats

# Информация о сервисах
curl http://localhost:3000/api/info
```

#### Redirect Service (http://localhost:3002)

```bash
# Перейти по короткому URL (перенаправление)
curl -L http://localhost:3002/{shortCode}
```

## 📊 Поток данных

```
1. Пользователь создаёт короткий URL:
   User → api-gateway → shortener-service → Redis

2. Пользователь переходит по короткому URL:
   User → redirect-service → Redis (получить URL)
                          ↓
                        Kafka (отправить событие клика)
                          ↓
                    analytics-service → MongoDB

3. Пользователь запрашивает статистику:
   User → api-gateway → analytics-service → MongoDB
```

## 🔧 Технологический стек

| Компонент | Технология | Версия |
|-----------|------------|--------|
| Backend | Go | 1.21 |
| API Gateway | Gorilla Mux, CORS | Latest |
| Message Broker | Apache Kafka | 7.5.0 |
| Coordination | Apache Zookeeper | 7.5.0 |
| Cache / Storage | Redis | 7 Alpine |
| Database | MongoDB | 7 |
| Distributed Tracing | Jaeger | 1.52 |
| Frontend | HTML, CSS, JavaScript | - |
| Web Server | Nginx | Alpine |
| Containerization | Docker, Docker Compose | Latest |

## 📖 Документация

- **[QUICKSTART.md](./QUICKSTART.md)** ⭐ - Быстрый старт за 3 минуты
- **[ARCHITECTURE.md](./ARCHITECTURE.md)** - Подробное описание архитектуры системы
- **[CI_CD.md](./CI_CD.md)** 🚀 - GitHub Actions с динамической матрицей
- **[MULTIPLATFORM.md](./MULTIPLATFORM.md)** 🌐 - Multi-platform builds (AMD64 + ARM64)
- **[JAEGER_GUIDE.md](./JAEGER_GUIDE.md)** 🔍 - Руководство по Distributed Tracing
- **[OPENTELEMETRY_EXAMPLE.md](./OPENTELEMETRY_EXAMPLE.md)** 🔧 - Пример добавления OpenTelemetry
- **[Makefile](./Makefile)** - Команды для управления проектом
- **[docker-compose.yml](./docker-compose.yml)** - Конфигурация всех сервисов

## 📦 Структура проекта

```
.
├── 📄 README.md                     # Основная документация
├── 📄 QUICKSTART.md                 # Быстрый старт за 3 минуты
├── 📄 ARCHITECTURE.md               # Подробная архитектура
├── 📄 CI_CD.md                      # 🆕 GitHub Actions workflows
├── 📄 PROJECT_OVERVIEW.md           # Обзор проекта
│
├── 🐳 docker-compose.yml            # Оркестрация всех сервисов
├── 📝 Makefile                      # Удобные команды
├── 🔧 .gitignore                    # Git ignore
├── 🔧 .dockerignore                 # Docker ignore
│
├── 📁 .github/                      # 🆕 GitHub Actions
│   └── workflows/                   # CI/CD workflows
│       ├── build-pr.yml             # Сборка PR с динамической матрицей
│       ├── build-main.yml           # Сборка main после мержа
│       └── build-all.yml            # Ручная сборка всех сервисов
│
├── 📁 .git-hooks/                   # 🆕 Git hooks
│   ├── prepare-commit-msg           # Автодобавление ID issue в коммиты
│   └── README.md                    # Документация git hooks

├── 📁 api-gateway/                  # Микросервис: API Gateway
│   ├── main.go                      # Go код (маршрутизация)
│   ├── go.mod                       # Go dependencies
│   └── Dockerfile                   # Multi-stage build
│
├── 📁 shortener-service/            # Микросервис: Создание коротких URL
│   ├── main.go                      # Go код (генерация + Redis)
│   ├── go.mod                       # Go dependencies
│   └── Dockerfile                   # Multi-stage build
│
├── 📁 redirect-service/             # Микросервис: Перенаправление
│   ├── main.go                      # Go код (Redis + Kafka Producer)
│   ├── go.mod                       # Go dependencies  
│   └── Dockerfile                   # Multi-stage build
│
├── 📁 analytics-service/            # Микросервис: Аналитика
│   ├── main.go                      # Go код (MongoDB + Kafka Consumer)
│   ├── go.mod                       # Go dependencies
│   └── Dockerfile                   # Multi-stage build
│
├── 📁 frontend/                     # Веб-интерфейс
│   ├── index.html                   # HTML + CSS + JS
│   └── Dockerfile                   # Nginx сервер
│
├── 📁 pkg/                          # Общие библиотеки
│   └── tracing/                     # OpenTelemetry трейсинг
│       ├── tracing.go               # Инициализация Jaeger
│       └── go.mod                   # Go dependencies
│
└── 📁 scripts/                      # Вспомогательные скрипты
    └── test-system.sh               # Полное тестирование системы
```

## 🧪 Тестирование

### Проверка здоровья сервисов

```bash
# API Gateway
curl http://localhost:3000/health

# Shortener Service
curl http://localhost:3001/health

# Redirect Service
curl http://localhost:3002/health

# Analytics Service
curl http://localhost:3003/health
```

### Полный сценарий

```bash
# 1. Создать короткий URL
RESPONSE=$(curl -s -X POST http://localhost:3000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://github.com"}')

echo $RESPONSE
# {"shortCode":"abc123","shortUrl":"http://localhost:3002/abc123","originalUrl":"https://github.com"}

# 2. Извлечь shortCode
SHORT_CODE=$(echo $RESPONSE | jq -r '.shortCode')

# 3. Перейти по короткой ссылке (несколько раз)
curl -L http://localhost:3002/$SHORT_CODE
curl -L http://localhost:3002/$SHORT_CODE
curl -L http://localhost:3002/$SHORT_CODE

# 4. Подождать немного (Kafka асинхронный)
sleep 2

# 5. Проверить статистику
curl http://localhost:3000/api/stats/$SHORT_CODE
# {"shortCode":"abc123","totalClicks":3,"lastClick":"2024-..."}
```

## 📈 Мониторинг

### Логи

```bash
# Все сервисы
docker-compose logs -f

# Конкретный сервис
docker-compose logs -f api-gateway
docker-compose logs -f redirect-service
docker-compose logs -f analytics-service
```

### Kafka топики

```bash
# Подключиться к Kafka контейнеру
docker exec -it url-shortener-kafka bash

# Просмотр сообщений в топике
kafka-console-consumer --bootstrap-server localhost:9092 \
  --topic url-clicks --from-beginning
```

### MongoDB

```bash
# Подключиться к MongoDB
docker exec -it url-shortener-mongodb mongosh

use analytics
db.clicks.find().pretty()
db.clicks.count()
```

### Redis

```bash
# Подключиться к Redis
docker exec -it url-shortener-redis redis-cli

KEYS url:*
GET url:abc123
```

## 🛑 Остановка

### С использованием Make

```bash
# Остановить все сервисы
make down

# Удалить все контейнеры и volumes
make clean

# Перезапустить
make restart
```

### С использованием Docker Compose

```bash
# Остановить все сервисы
docker-compose down

# Остановить и удалить volumes (очистить данные)
docker-compose down -v
```

## 🔥 Масштабирование

### С использованием Make

```bash
# Масштабировать analytics-service до 3 инстансов
make scale-analytics
```

### С использованием Docker Compose

```bash
# Запустить несколько инстансов analytics-service
docker-compose up -d --scale analytics-service=3

# Kafka автоматически распределит нагрузку между consumer'ами в группе
```

### Как это работает

- **API Gateway** - можно масштабировать, поставить за load balancer
- **Shortener Service** - stateless, легко масштабируется
- **Redirect Service** - stateless, масштабируется горизонтально
- **Analytics Service** - Kafka Consumer Group автоматически распределяет партиции между инстансами

## 💡 Идеи для расширения

- [ ] **Добавить OpenTelemetry код** - см. [OPENTELEMETRY_EXAMPLE.md](./OPENTELEMETRY_EXAMPLE.md)
- [ ] Добавить аутентификацию (JWT)
- [ ] Добавить rate limiting
- [ ] Добавить кэширование статистики
- [ ] Добавить TTL для коротких URL
- [ ] Добавить custom aliases
- [ ] Добавить QR коды
- [ ] Добавить Prometheus + Grafana для мониторинга
- [ ] Добавить Kubernetes манифесты
- [ ] Добавить CI/CD с GitHub Actions
- [ ] Добавить тесты (unit, integration, e2e)

## 📝 Лицензия

MIT

## 👨‍💻 Автор

Демонстрационный проект для изучения микросервисной архитектуры
