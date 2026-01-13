# **Техническое задание: Feed-сервис**

## **1. Цель проекта**

Создать высоконагруженный backend-сервис ленты (feed) на PHP (Laravel/Symfony) с:

* CQRS (разделение чтения и записи)
* PostgreSQL (write + read model)
* Redis (кэш)
* RabbitMQ/Kafka (очередь)
* SSO (OAuth2 / OpenID Connect)
* Поддержкой медиа (изображений постов и аватарок) через S3/MinIO
* Полной системой автотестов (unit, integration, E2E)
* Возможностью локально симулировать миллион пользователей и нагрузку
* Быстрым локальным разворачиванием всех сервисов и тестов одной командой через Taskfile
* Мониторингом через Prometheus + Grafana

---

## **2. Функциональные требования**

### **2.1 Пользователи**

* Аутентификация через SSO (OAuth2/OpenID Connect)
* Минимум один внешний провайдер (Google/GitHub)
* JWT для API, refresh token для долгосрочного использования
* Создание пользователя при первом входе через SSO
* Подписка/отписка на других пользователей
* Получение персонального feed

### **2.2 Посты**

* Создание поста (текст + опционально изображение)
* Лайк / анлайк поста
* Пост immutable (редактирование не требуется)
* Все операции проходят через Command-часть CQRS
* Idempotency ключи для команд

### **2.3 Feed**

* Персональная лента пользователя
* Сортировка по времени (desc)
* Пагинация / infinite scroll
* Eventual consistency допускается
* Быстрая выдача через read-model и Redis cache

### **2.4 Медиа / изображения**

* Возможность загружать изображения постов и аватарок пользователей
* Валидация формата (jpg, png, webp) и размера (<5MB)
* Хранение на S3 или локально через MinIO
* Bucket структура:

  ```
  /users/{user_id}/avatars/{filename}
  /posts/{post_id}/{filename}
  ```
* Прямые ссылки через API или presigned URL
* Удаление старых изображений при обновлении поста или аватара
* Возможность асинхронной обработки изображений через Worker (например, миниатюры, оптимизация)

---

## **3. Архитектурные требования**

### **3.1 CQRS**

* **Write Side (Command)**: `CreatePostCommand`, `LikePostCommand`, `FollowUserCommand`, `UploadPostImageCommand`
* **Event**: `PostCreatedEvent`, `PostLikedEvent`, `UserFollowedEvent`, `PostImageUploadedEvent`
* **Read Side (Query)**: таблица `user_feed` + Redis cache
* **Worker**: асинхронная обработка событий, обновление read-model и кэша, retry/backoff, dead-letter queue, idempotency

### **3.2 База данных**

* PostgreSQL (write + read model)
* Основные таблицы: `users`, `posts`, `follows`, `likes`, `user_feed`
* Read-model оптимизирована под быстрые SELECT
* Логическое шардирование пользователей и постов
* Отдельная тестовая база для автотестов

### **3.3 Очереди**

* RabbitMQ или Kafka
* Fan-out при публикации постов
* Retry, backoff, dead-letter queue
* Worker обрабатывает события асинхронно

### **3.4 Кэш**

* Redis
* Кэш последних N постов пользователя
* Кэш популярных постов
* TTL и инвалидация
* Опционально: rate limiting

### **3.5 Авторизация (SSO)**

* OAuth2 / OpenID Connect
* JWT / Access token
* Refresh token
* Возможность мокать SSO для тестов

### **3.6 Медиа / MinIO**

* Локальное S3-совместимое хранилище через MinIO
* Конфиг через `.env`:

  ```
  S3_ENDPOINT=http://minio:9000
  S3_KEY=minioadmin
  S3_SECRET=minioadmin
  S3_BUCKET=app-bucket
  S3_REGION=us-east-1
  ```

---

## **4. Тестирование**

### **4.1 Unit**

* Модели (`User`, `Post`, `Follow`, `Like`)
* Command/Query сервисы

### **4.2 Integration**

* API + PostgreSQL + Redis + Queue + Worker
* Проверка CQRS pipeline: Command → Event → Worker → Read-model → Query

### **4.3 Functional / E2E**

* SSO login → создание пользователя
* Создание поста → обновление feed
* Загрузка изображения → выдача presigned URL

### **4.4 Load testing**

* GET `/feed`, POST `/posts`, POST `/likes`
* Метрики: latency (p95/p99), RPS, error rate, queue backlog, cache hit/miss
* Запуск нагрузочного теста через Docker (k6/JMeter)

---

## **5. Локальный запуск / Developer Experience**

### **5.1 Быстрый старт**

* Поднятие всей инфраструктуры одной командой (`task up`) через Docker Compose:

    * API, PostgreSQL, Redis, Queue, Worker, Prometheus, Grafana, MinIO, Load-generator

### **5.2 Taskfile команды**

```yaml
tasks:
  up: docker-compose up -d
  down: docker-compose down
  seed: docker exec app php artisan db:seed
  test:unit: docker exec app vendor/bin/phpunit tests/Unit
  test:integration: docker exec app vendor/bin/phpunit tests/Integration
  test:e2e: docker exec app vendor/bin/phpunit tests/E2E
  test:load: docker exec load-generator k6 run /scripts/load-test.js
```

---

## **6. Мониторинг**

* **Prometheus**: сбор метрик с API, Worker, Redis
* **Grafana**: визуализация latency, RPS, queue backlog, cache hit/miss
* Dashboard для мониторинга производительности и нагрузочных тестов

---

## **7. Нефункциональные требования**

* Быстрое чтение feed через read-model + Redis
* Минимизация JOIN-ов на горячих запросах
* Eventual consistency допустима
* Масштабируемость: возможность логического шардирования и горизонтального масштабирования Worker’ов и API

---

## **8. Tech Stack**

* PHP 8+, Laravel или Symfony
* PostgreSQL
* Redis
* RabbitMQ / Kafka
* OAuth2 / OpenID Connect (SSO)
* MinIO / S3
* Docker / Docker Compose
* Taskfile
* PHPUnit / Pest
* k6 / JMeter
* Prometheus + Grafana

---

## **9. Результат**

* Полностью рабочее feed-приложение
* CQRS с Command/Query и Worker
* Redis-кэш + PostgreSQL read-model
* SSO авторизация
* Медиа-поддержка через MinIO/S3
* Полное покрытие автотестами
* Локальная нагрузочная генерация и мониторинг через Prometheus/Grafana
* Все сервисы поднимаются одной командой (`task up`)