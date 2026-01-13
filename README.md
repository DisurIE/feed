
# Техническое задание

## **1. Цель проекта**

Создать высоконагруженный backend-сервис ленты (feed) на PHP (Laravel/Symfony) с:

* CQRS (разделение чтения и записи),
* PostgreSQL (write + read модель),
* Redis (кэш),
* RabbitMQ/Kafka (очереди),
* SSO (OAuth2 / OpenID Connect),
* полной системой автотестов,
* возможностью локально симулировать миллион пользователей и нагрузку,
* быстрым локальным разворачиванием и запуском всех тестов одной командой через Taskfile.

---

## **2. Функциональные требования**

### **2.1 Пользователи**

* Аутентификация через SSO (OAuth2/OpenID Connect)
* Поддержка минимум одного внешнего провайдера (Google/GitHub)
* JWT для авторизации API
* Refresh token для долгосрочного использования
* Создание пользователя при первом входе через SSO
* Подписка/отписка на других пользователей
* Получение персонального feed

### **2.2 Посты**

* Создание поста (текст)
* Лайк / анлайк поста
* Пост immutable (редактирование не требуется)
* Все операции проходят через Command-часть CQRS
* Идентификаторы команд имеют idempotency ключи

### **2.3 Feed**

* Персональная лента пользователя
* Сортировка по времени (desc)
* Пагинация / infinite scroll
* Eventual consistency допускается
* Быстрая выдача через read-model и Redis cache

---

## **3. Архитектурные требования**

### **3.1 CQRS**

* **Write Side (Command)**

    * `CreatePostCommand`, `LikePostCommand`, `FollowUserCommand`
    * Source of truth — write-таблицы PostgreSQL
* **Events**

    * `PostCreatedEvent`, `PostLikedEvent`, `UserFollowedEvent`
    * Асинхронная обработка воркером
* **Read Side (Query)**

    * Таблица `user_feed` для быстрого чтения
    * Redis cache для последних N постов
* **Worker**

    * Асинхронная обработка событий
    * Обновление read-model и кэша
    * Retry с backoff, dead-letter queue, idempotency

### **3.2 База данных**

* PostgreSQL для write и read моделей
* Основные таблицы:

    * `users`, `posts`, `follows`, `likes`, `user_feed`
* Read-model оптимизирована под быстрые SELECT
* Возможность логического шардирования пользователей
* Отдельная тестовая база PostgreSQL для автотестов

### **3.3 Очереди**

* RabbitMQ или Kafka
* Fan-out при публикации постов
* Retry, backoff, dead-letter queue
* Обработка всех событий через воркеры

### **3.4 Кэш**

* Redis
* Кэш последних N постов пользователя
* Кэш популярных постов
* TTL и логика инвалидации
* Возможность rate limiting

---

## **4. Тестирование**

### **4.1 Unit**

* Бизнес-логика Command и Query сервисов
* Модели: User, Post, Follow, Like

### **4.2 Integration**

* API + PostgreSQL + Redis + Queue + Worker
* Проверка CQRS pipeline: Command → Event → Read-model → Query

### **4.3 Functional / E2E**

* SSO login → создание пользователя
* Создание поста → обновление feed подписчиков
* Получение feed через API

### **4.4 Load testing**

* Симуляция:

    * GET `/feed` (основной сценарий)
    * POST `/posts` (fan-out)
    * POST `/likes`
* Метрики: latency (p95/p99), RPS, error rate, queue backlog, cache hit/miss
* Запуск нагрузочного теста через Docker контейнер, интегрированный с Taskfile

---

## **5. Локальный запуск / Developer Experience**

### **5.1 Быстрый старт**

* Полное разворачивание инфраструктуры одной командой (`task up`)
* Все сервисы поднимаются через Docker Compose:

    * PHP API
    * PostgreSQL
    * Redis
    * RabbitMQ/Kafka
    * Worker
    * Load generator (k6/JMeter)

### **5.2 Taskfile команды**

* `task up` — поднять всю инфраструктуру
* `task down` — остановить
* `task seed` — загрузка тестовых и больших данных
* `task test` — запуск всех тестов
* `task test:unit` — unit-тесты
* `task test:integration` — integration-тесты
* `task test:e2e` — functional / E2E
* `task test:load` — нагрузочное тестирование
* Возможность запускать нагрузку в Docker, без конфликтов с локальной средой

---

## **6. Нефункциональные требования**

### **6.1 Производительность**

* Быстрая выдача feed через read-model и Redis
* Минимизация JOIN-ов
* Eventual consistency допустима

### **6.2 Масштабируемость**

* Легкое логическое шардирование
* Возможность горизонтального масштабирования Worker’ов и API

### **6.3 Мониторинг (опционально для senior)**

* Метрики:

    * Worker processing time
    * Queue backlog
    * Cache hit/miss
    * Latency и RPS API
* Возможность интеграции с Prometheus/Grafana или вывод логов в JSON для анализа

---

## **7. Tech Stack**

* Backend: PHP 8+, Laravel или Symfony
* DB: PostgreSQL
* Cache: Redis
* Queue: RabbitMQ / Kafka
* Auth: OAuth2 / OpenID Connect
* Docker / Docker Compose
* Taskfile для управления локальной средой и тестами
* Testing: PHPUnit / Pest
* Load testing: k6 / JMeter (Docker)
* Optional: Prometheus/Grafana для мониторинга

---

## **8. Результат**

* Полностью рабочее feed-приложение
* CQRS с разделением Command / Query
* Асинхронная обработка через очереди и Worker
* Redis-кэш и PostgreSQL read-model
* SSO-авторизация
* Полное покрытие автотестами (unit, integration, E2E)
* Возможность локально запускать нагрузку и тесты одной командой
* Документация архитектуры и trade-offs

---