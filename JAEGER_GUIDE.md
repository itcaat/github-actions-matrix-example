# 🔍 Jaeger - Distributed Tracing

## Что это?

**Jaeger** - это инструмент для отслеживания запросов в микросервисной архитектуре.

## 📊 Что вы увидите

### 1. Полный путь запроса
```
User Request
  └─ api-gateway (5ms)
      └─ shortener-service (10ms)
          └─ Redis (2ms)
```

### 2. Временные метрики
- Общее время запроса: 17ms
- Время в каждом сервисе
- Время на I/O операции (Redis, MongoDB, Kafka)

### 3. Асинхронные операции
```
Redirect Request
  └─ redirect-service (3ms)
      ├─ Redis GET (1ms)
      └─ Kafka Publish (async)
          └─ analytics-service (5ms)
              └─ MongoDB INSERT (2ms)
```

## 🚀 Как использовать

### Запуск

Jaeger уже включён в docker-compose:

```bash
docker-compose up -d
```

### Открыть UI

Откройте в браузере: **http://localhost:16686**

### Просмотр трейсов

1. **Выберите сервис** из выпадающего списка:
   - `api-gateway`
   - `shortener-service`
   - `redirect-service`
   - `analytics-service`

2. **Нажмите "Find Traces"**

3. **Кликните на трейс** чтобы увидеть детали

## 📖 Основные концепции

### Trace (Трейс)
Полный путь одного запроса через всю систему.

### Span (Спан)
Одна операция внутри трейса (например, HTTP запрос или запись в БД).

### Tags (Теги)
Метаданные спана:
- `http.method=POST`
- `http.status_code=200`
- `db.type=redis`

### Logs (Логи)
События внутри спана с временными метками.

## 🔍 Примеры использования

### Сценарий 1: Отладка медленного запроса

1. Создайте несколько коротких URL
2. Откройте Jaeger UI
3. Найдите медленный трейс
4. Посмотрите, какой сервис тормозит

**Что искать:**
- Большие временные промежутки между спанами
- Долгие операции с БД
- Сетевые задержки

### Сценарий 2: Поиск ошибок

1. Создайте невалидный запрос
2. В Jaeger найдите трейсы с ошибками (красные)
3. Посмотрите, на каком сервисе произошла ошибка
4. Изучите теги и логи для деталей

### Сценарий 3: Визуализация потока

1. Создайте короткий URL
2. Перейдите по нему несколько раз
3. Посмотрите статистику
4. В Jaeger найдите все связанные трейсы

**Вы увидите:**
- Синхронный поток: `api-gateway → shortener-service → Redis`
- Асинхронный поток: `redirect-service → Kafka → analytics-service → MongoDB`

## 🎯 Полезные фильтры

### По сервису
```
service=api-gateway
```

### По операции
```
operation=POST /api/shorten
```

### По времени выполнения
```
minDuration=100ms
```

### По тегам
```
http.status_code=500
```

## 📈 Метрики которые вы увидите

### Latency (Задержка)
- P50 (медиана): 50% запросов быстрее этого значения
- P95: 95% запросов быстрее
- P99: 99% запросов быстрее

### Throughput (Пропускная способность)
- Requests per second
- Requests per minute

### Error Rate (Процент ошибок)
- Количество failed spans
- Процент успешных запросов

## 🔧 Архитектура трейсинга

### Как это работает

1. **Instrumentation**
   - Каждый сервис создаёт spans для операций
   - Spans содержат timing и metadata

2. **Context Propagation**
   - Trace ID передаётся через HTTP headers
   - Все spans одного запроса имеют один Trace ID

3. **Collection**
   - Spans отправляются в Jaeger Agent (UDP 6831)
   - Agent отправляет в Jaeger Collector

4. **Storage**
   - Трейсы хранятся in-memory (в all-in-one режиме)
   - Для production используйте Cassandra или Elasticsearch

5. **Query**
   - UI запрашивает данные через Jaeger Query API
   - Визуализация в браузере

### Компоненты Jaeger

```
┌─────────────────┐
│  Jaeger Agent   │ ← Принимает spans от сервисов (UDP)
└────────┬────────┘
         ↓
┌─────────────────┐
│ Jaeger Collector│ ← Обрабатывает и сохраняет
└────────┬────────┘
         ↓
┌─────────────────┐
│  Jaeger Storage │ ← In-memory / Cassandra / ES
└────────┬────────┘
         ↓
┌─────────────────┐
│  Jaeger Query   │ ← API для UI
└────────┬────────┘
         ↓
┌─────────────────┐
│   Jaeger UI     │ ← http://localhost:16686
└─────────────────┘
```

## 🌐 Порты Jaeger

| Порт | Протокол | Назначение |
|------|----------|------------|
| 5775 | UDP | zipkin.thrift |
| 6831 | UDP | jaeger.thrift compact (используется сервисами) |
| 6832 | UDP | jaeger.thrift binary |
| 5778 | HTTP | Serve configs |
| 16686 | HTTP | **Web UI** ⭐ |
| 14268 | HTTP | Collector API |
| 14250 | gRPC | Collector gRPC |
| 9411 | HTTP | Zipkin compatible endpoint |

## 💡 Tips & Tricks

### 1. Сравнение трейсов
- Откройте два трейса в разных вкладках
- Сравните timing
- Найдите аномалии

### 2. Поиск по trace ID
- Скопируйте Trace ID из логов
- Вставьте в Jaeger UI
- Найдите конкретный запрос

### 3. Группировка по операциям
- Смотрите статистику по endpoint'ам
- Находите самые медленные операции

### 4. Timeline view
- Визуализация параллельных операций
- Видно, что выполняется одновременно

## 🐛 Troubleshooting

### Не видно трейсов

**Проверьте:**
1. Jaeger запущен: `docker-compose ps jaeger`
2. Сервисы отправляют данные: проверьте логи
3. Правильный временной диапазон в UI

### Неполные трейсы

**Причины:**
- Сервис не инструментирован (не добавлен OpenTelemetry код)
- Context не передаётся между сервисами
- Spans не flush'атся (не отправляются)

### Jaeger UI не открывается

```bash
# Проверить статус
docker logs url-shortener-jaeger

# Перезапустить
docker-compose restart jaeger
```

## 📚 Дополнительные ресурсы

- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [OpenTelemetry](https://opentelemetry.io/)
- [Distributed Tracing Best Practices](https://www.jaegertracing.io/docs/latest/best-practices/)

## 🎓 Что изучить дальше

1. **Sampling Strategies**
   - Probabilistic (вероятностный)
   - Rate limiting
   - Adaptive sampling

2. **Production Setup**
   - Separate Jaeger components
   - Persistent storage (Cassandra/Elasticsearch)
   - High availability

3. **Advanced Instrumentation**
   - Custom spans
   - Span events
   - Baggage (контекст между сервисами)

4. **Integration**
   - Prometheus metrics
   - Grafana dashboards
   - Alerting на базе трейсов

---

**Автор:** URL Shortener Demo  
**Версия:** 1.0 with Jaeger  

Приятного трейсинга! 🔍
