# 🚀 Быстрый старт

## За 3 минуты до работающей системы

### Шаг 1: Клонирование (если ещё не сделано)

```bash
git clone <repo-url>
cd github-actions-matrix-example
```

### Шаг 2: Запуск

```bash
# Один из вариантов:

# Вариант A: С Make (удобнее)
make up-build

# Вариант B: Без Make
docker-compose up -d --build
```

**Подождите 30-60 секунд**, пока все сервисы запустятся.

### Шаг 3: Проверка

Откройте в браузере: **http://localhost:8080**

🎉 **Готово!**

---

## Первые шаги

### 1. Создайте короткий URL

В веб-интерфейсе:
1. Введите любой URL (например, `https://github.com`)
2. Нажмите "Сократить URL"
3. Скопируйте сокращённую ссылку

### 2. Используйте короткий URL

Откройте сокращённую ссылку в новой вкладке. Вы будете перенаправлены на оригинальный сайт.

### 3. Посмотрите статистику

1. Вернитесь на http://localhost:8080
2. Нажмите "Обновить статистику"
3. Увидите количество переходов по вашей ссылке

---

## Проверка через API

### Создать короткий URL

```bash
curl -X POST http://localhost:3000/api/shorten \
  -H "Content-Type: application/json" \
  -d '{"url": "https://example.com"}'
```

**Ответ:**
```json
{
  "shortCode": "abc123",
  "shortUrl": "http://localhost:3002/abc123",
  "originalUrl": "https://example.com"
}
```

### Перейти по ссылке

```bash
curl -L http://localhost:3002/abc123
```

### Получить статистику

```bash
curl http://localhost:3000/api/stats/abc123
```

**Ответ:**
```json
{
  "shortCode": "abc123",
  "totalClicks": 5,
  "lastClick": "2024-01-01T12:34:56Z"
}
```

---

## Полезные команды

### С Make

```bash
# Посмотреть все команды
make help

# Проверить здоровье сервисов
make health

# Посмотреть логи
make logs

# Остановить
make down

# Удалить всё
make clean
```

### Без Make

```bash
# Статус сервисов
docker-compose ps

# Логи всех сервисов
docker-compose logs -f

# Логи конкретного сервиса
docker-compose logs -f api-gateway

# Остановить
docker-compose down

# Удалить с данными
docker-compose down -v
```

---

## Troubleshooting

### Проблема: Сервисы не запускаются

**Решение:**
```bash
# Проверить логи
docker-compose logs

# Перезапустить
docker-compose down
docker-compose up -d --build
```

### Проблема: Порты заняты

**Решение:** Измените порты в `docker-compose.yml`

### Проблема: Медленная работа

**Решение:** Дайте больше ресурсов Docker (Settings → Resources)

---

## Что дальше?

1. **Изучите архитектуру** - прочитайте [ARCHITECTURE.md](./ARCHITECTURE.md)
2. **Посмотрите код** - изучите Go-сервисы
3. **Экспериментируйте** - измените код, пересоберите
4. **Масштабируйте** - `make scale-analytics`
5. **Добавьте функции** - смотрите "Идеи для расширения" в README

---

## Полезные ссылки

- **Frontend:** http://localhost:8080
- **API Gateway:** http://localhost:3000/health
- **Shortener Service:** http://localhost:3001/health
- **Redirect Service:** http://localhost:3002/health
- **Analytics Service:** http://localhost:3003/health

---

🎓 **Учебные цели:**
- Понять микросервисную архитектуру
- Освоить Docker Compose
- Изучить Kafka для асинхронного взаимодействия
- Понять разделение хранилищ (Redis, MongoDB)
- Применить паттерн API Gateway

Удачи в изучении! 🚀
