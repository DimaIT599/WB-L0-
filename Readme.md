# Тестовое задание L0 учебного курса 'Горутиновый golang' от техношколы WB

**Демонстрационный сервис с Kafka, PostgreSQL, кешем**

## Содержание

- [Использованные технологии](#used_tech)
- [Особенности реализации](#important_features)
- [Архитектура сервиса](#arch)
- [Структура проекта](#project_struct)
- [Установка и запуск](#installation_and_launch)



<a name="used_tech"><h2>Использованные технологии</h2></a>

* **Язык проекта**: Golang
    * **Логгер**:  zap logger(от Uber)
    * **Роутер**: HTTP роутер mux(от gorilla) 
    * **Горутины**
    * **gracefull shutdown**
* **БД**: PostgreSQL
* **Кэш**: Redis
* **Брокер сообщений**: Kafka
* **Визуализация состояния топиков Kafka**: Kafka UI
* **Документирование HTTP APi**: Swagger(https://github.com/swaggo)
* **Виртуализация**: Docker(Docker-compose)



<a name="arch"><h2>Архитектура сервиса</h2></a>

Приложение представляет сервис, обеспечивающий следующую требуемую в задании функциональность. 

Основные модули(*я пытался в 'Чистую архитектуру' и разделение ответственности между слоями-модулями*):

* Kafka-консьюмер(количество консьюмеров регулируется через переменные окружения).
* Kafka-хендлер - выполняет функцию валидатора входящих сообщений и передачу в сервисный слой(и прием-передачу результатов из сервисного слоя в Kafka-консьюмер).
* HTTP-сервер с шлюзом и маршрутизатором для обработки HTTP-запросов(вызова соответствующих HTTP-хендреров).
* HTTP-хендреры для вызова соответствующих методов сервисного слоя и обработки результатов.
* Сервисный слой - обрабатывает входящие запросы от хендлеров, отправляет соответствующие ответы, взаимодействует с кэшем и репозиторием приложения.
* Слой репозитория - обрабатывает запросы сервисного слоя и взаимодействует с БД.

```
flowchart LR
    K[Kafka] -->|Сообщения| C[Consumer]
    C --> S[Service Layer]
    S -->|Сохранение| DB[(PostgreSQL)]
    S -->|Кэширование| R[(Redis)]
    UI[HTTP API / Swagger / Frontend] --> S
```


<a name="project_struct"><h2>Структура проекта</h2></a>

```
.
├── cmd
│   ├── main
│   │   └── main.go           - точка входа
│   ├── order-producer
│   │   └── main.go           - генератор сообщений(заказов) для Kafka
│   └── tools
│       └── create_dlq_topic
│           └── main.go       - создание DLQ топика Kafka
├── docker-compose.yaml       - конфигурация сборки Docker-контейнеров внешних компонетов сервиса
├── docs
│   ├── docs.go
│   ├── swagger.json
│   └── swagger.yaml       - swagger документация HTTP API
├── frontend
│   ├── app.js
│   └── index.html         - фронтенд интерфейс для получения заказов по UID
├── go.mod
├── go.sum
├── internal
│   ├── app
│   │   └── app.go         - файл инициализации моделей приложения
│   ├── cache
│   │   └── cache.go       - методы кэша
│   ├── config
│   │   └── config.go        - конфигурация приложения      
│   ├── delivery
│   │   ├── http
│   │   │   ├── handler.go       - HTTP хендлеры
│   │   │   ├── handler_test.go  - .unit-тесты для HTTP хендлеров
│   │   │   └── helper.go        - вспомогательные функции HTTP хендлеров
│   │   └── kafkadelivery
│   │       ├── consumer.go      - код консьюмера(читателя) Kafka
│   │       ├── errors.go        - кастомные ошибки пакета для консьюмера
│   │       ├── event.go         - схема и функция валидации входящего сообщения
│   │       └── handler.go       - Kafka хендлер
│   ├── dto
│   │   └── dto.go               - модели, доступные хендлерам(HTTP хендлеры - для перемаппинга моделей сервиса)
│   ├── gateway
│   │   ├── gateway.go           -  HTTP-сервер
│   │   └── routes.go            - маршрутизатор HTTP-сервера
│   ├── orders
│   │   ├── errors.go            - ошибки домена заказов
│   │   └── models.go            - модели домена заказов
│   ├── repository
│   │   └── repository.go        - репозиторий для обработки запросов от сервиса обработки заказов
│   └── service
│       ├── orders_cache.go         - декларация интерфейсов кэша для сервиса
│       ├── orders_helpers.go       - хелперы для сервисного слоя
│       ├── orders_repository.go    - декларация интерфейсов для репозитория
│       ├── orders_service.go       - декларация публичных интерфейсов сервиса обработки заказов
│       ├── orders_service_impl.go  - имплементация функций сервисного слоя
│       └── orders_service_test.go  - unit-тесты для сервисного слоя
├── Makefile      - скрипты автоматизации
├── migrations
│   └── 001_create_order_tables.sql - скрипт создания структур таблиц БД(модель данных для PostgreSQL)
├── pkg
│   ├── db
│   │   └── postgres.go    - инициализатор подключения к PostgreSQL
│   ├── logger
│   │   └── logger.go      - обертка над zap, логгер сервиса 
│   └── redisclient
│       └── redisclient.go - инициализатор подключения клиента Redis
└── Readme.md

```


<a name="installation_and_launch"><h2>Установка и запуск</h2></a>


1. Склонируйте репозиторий. 

```
git clone https://github.com/idalgo-2021/wb_tech_level_zero.git
```

2. Установите необходимые зависимости(выполнить из корня проекта).

```
go mod tidy
```

3. Подготовьте '.env' - файл.

Создайте в корне проекта файл '.env' и заполните его по примеру '.env.example'(можно скопировать его содержимое, оно также скорее всего будет рабочим).  

4. Создайте необходимые для работы приложения Docker-контейнеры(выполнить в корне проекта).

```
make up
```

5. Создайте основной топик Kafka внутри контейнера(выполнить в корне проекта команды).

```
sudo docker exec -it kafka /bin/bash


kafka-topics.sh --create --topic orders --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1
```

6. Создайте топик kafka для DLQ(выполнить в корне проекта).

```
go run ./cmd/tools/create_dlq_topic
```

7. Запустите приложение(выполнить в корне проекта).

```
go run ./cmd/main/main.go
```
Пример лога при полностью успешном старте:

```
mikha@basta3:~/my_development/my_go/wb_tech/wb_tech_level_zero$ go run ./cmd/main/main.go
{"level":"info","ts":1755341014.0010533,"caller":"main/main.go:42","msg":"Configuration loaded"}
{"level":"info","ts":1755341014.0237105,"caller":"logger/logger.go:46","msg":"Starting HTTP server on 127.0.0.1:10000","service":"wb_tech"}
{"level":"info","ts":1755341014.0237777,"caller":"logger/logger.go:46","msg":"HTTP server is starting to listen","service":"wb_tech"}
{"level":"info","ts":1755341014.023764,"caller":"logger/logger.go:46","msg":"Warming up Redis order cache...","service":"wb_tech"}
{"level":"info","ts":1755341014.0237963,"caller":"logger/logger.go:46","msg":"Starting Kafka consumer...","service":"wb_tech"}
{"level":"info","ts":1755341014.0239084,"caller":"logger/logger.go:46","msg":"Worker 1 started","service":"wb_tech"}
{"level":"info","ts":1755341014.0238338,"caller":"logger/logger.go:46","msg":"Warming up cache...","service":"wb_tech"}
{"level":"info","ts":1755341014.0239651,"caller":"logger/logger.go:46","msg":"Worker 0 started","service":"wb_tech"}
{"level":"info","ts":1755341014.0356164,"caller":"logger/logger.go:46","msg":"Cache warmup completed","service":"wb_tech","count":2,"failed":0}
```

8. Для генерации и отправки сообщения в Kafka(выполнить в корне проекта).

```
go run cmd/order-producer/main.go
```

9. Пример использования рабочих ресурсов(ручек) при успешном запуске приложения:

```
   # /order/order_uid - основная ручка по заданию(документирована в swagger)
   http://localhost:10000/order/b563feb7b2b84b6test

   # /orders - дополнительная ручка, возвращающая список заказов(недокументирована в swagger; данные запрашиваются напрямую из БД, без использования кэша)
   http://localhost:10000/orders

   # /swagger/index.html - swagger описание HTTP API приложения в формате OpenAPI
   http://localhost:10000/swagger/index.html

   # [port]/ui/ - ручка Kafka UI, позволяющая визуализировать состояние топиков Kafka
   http://localhost:8080/ui/

   # После генерации сообщений(заказов) в кафку и записи их в БД, по их UID у вас так же должен работать и фронтенд `frontend/index.html`

```

