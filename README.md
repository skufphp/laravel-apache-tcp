# Laravel Apache TCP Lab

Тестовый проект для проверки сборки и запуска Laravel на базе связки Apache (Httpd) + PHP-FPM через TCP.

Репозиторий используется как лабораторный стенд для Laravel-приложения, собранного на связке:

- PHP-FPM + Apache (Httpd) через TCP
- PostgreSQL
- Redis
- Node.js / Vite
- Docker Compose для локальной разработки, локального production-like запуска и server-side production compose

Отдельно проект подготовлен так, чтобы его было удобно использовать для деплоя через Dokploy: production-конфигурация уже вынесена в отдельные compose-файлы и ориентирована на контейнерный запуск.

## Что здесь проверяется

- Сборка Laravel-приложения с использованием Apache в качестве веб-сервера
- Работа Apache и PHP-FPM через TCP внутри Docker-сети
- Dev-режим с примонтированным кодом, Vite и Xdebug
- Production-сборка с multi-stage Dockerfile (`php.Dockerfile` и `httpd.Dockerfile`)
- Запуск очередей и scheduler в отдельных контейнерах
- Отдельные production compose-файлы для локальной проверки и серверного деплоя
- Готовность проекта к деплою через Docker Compose / Dokploy

## Архитектура проекта

- **Apache (Httpd)** проксирует запросы в **PHP-FPM** через TCP (`laravel-php-apache-tcp:9000`)
- **PostgreSQL** используется как основная БД
- **Redis** используется для кеша, сессий и очередей
- **Node-контейнер** отвечает за frontend-сборку и Vite HMR в режиме разработки
- **Production-сборка** выполняется через multi-stage Dockerfile для PHP и Apache

## Структура

- `docker/` - Dockerfiles и конфигурация PHP / Apache (Httpd)
- `docker-compose.yml` - окружение для разработки
- `docker-compose.prod.local.yml` - локальный запуск production-профиля с публикацией `HTTPD_PORT`
- `docker-compose.prod.yml` - production-конфигурация для серверного деплоя без проброса локальных портов
- `Makefile` - основные команды для разработки и обслуживания
- `.env.example` - шаблон переменных для dev
- `.env.production.example` - шаблон переменных для production

## Быстрый старт для разработки

1. Скопируйте переменные окружения:

```bash
cp .env.example .env
```

2. Заполните `.env` реальными значениями.

Минимально проверьте:

- `APP_*`
- `DB_*`
- `REDIS_*`
- `HTTPD_PORT` (порт, на котором будет доступен сайт)
- `DB_FORWARD_PORT`
- `REDIS_FORWARD_PORT`
- `PGADMIN_PORT`
- `XDEBUG_*` при необходимости

3. Запустите инициализацию проекта:

```bash
make setup
```

Команда:

- соберет dev-образы
- поднимет контейнеры
- дождется готовности PostgreSQL и Redis
- установит Composer и NPM зависимости
- сгенерирует `APP_KEY`
- выполнит миграции
- выставит права на `storage/` и `bootstrap/cache`
- удалит `public/.htaccess` (не используется, так как конфигурация Apache встроена в образ)

После запуска будут доступны:

- Laravel: `http://localhost:8050` по умолчанию (порт берется из `HTTPD_PORT`)
- Vite dev server: `http://localhost:5173`
- pgAdmin: `http://localhost:8080` по умолчанию (порт берется из `PGADMIN_PORT`)

Если нужна ручная последовательность, используйте:

```bash
make build
make up
make install-deps
make artisan CMD="key:generate"
make migrate
make cleanup-httpd
```

## Основные команды

### Development

```bash
make up           # Запустить контейнеры (Dev)
make down         # Остановить контейнеры
make restart      # Перезапустить контейнеры
make build        # Собрать образы
make rebuild      # Пересобрать образы без кэша
make logs         # Показать логи всех сервисов
make logs-php     # Логи PHP-FPM
make logs-httpd   # Логи Apache
make status       # Статус контейнеров
make info         # Информация о портах и сервисах
```

### Laravel / PHP / Node

```bash
make artisan CMD="migrate"
make composer CMD="install"
make migrate
make rollback
make fresh        # Пересоздать базу и запустить сиды
make tinker
make test-php     # Запустить тесты (PHPUnit)
make npm-install
make npm-build    # Собрать фронтенд
```

### Shell и CLI-доступ

```bash
make shell-php
make shell-httpd
make shell-node
make shell-postgres   # Вход в psql
make shell-redis      # Проверка связи с Redis
```

### Production / Production-like

```bash
make up-prod          # Запуск локального prod-окружения
make down-prod        # Остановка prod-окружения
make rebuild-prod     # Пересборка prod-образов
make logs-prod        # Логи всех prod-сервисов
make shell-php-prod   # Shell в prod PHP контейнере
```

## Production-like локальный запуск

Для локальной проверки production-сценария:

1. Скопируйте шаблон:

```bash
cp .env.production.example .env.production
```

2. Заполните `.env.production`.

3. Запустите production-профиль:

```bash
make up-prod
```

Этот target использует:

- `.env.production`
- `docker-compose.prod.local.yml`
- production stage из `docker/php.Dockerfile` и `docker/httpd.Dockerfile`

В production-профиле:

- используется production stage образов
- Laravel собирается без dev-зависимостей
- frontend-ассеты собираются на этапе сборки образа
- миграции выполняются автоматически при старте PHP-контейнера
- выполняется `php artisan optimize:clear`
- queue worker и scheduler вынесены в отдельные сервисы
- локально публикуется только порт Apache, заданный через `HTTPD_PORT`

## Dokploy

Проект полностью готов для деплоя через Dokploy как Docker Compose приложение.

- Используйте `docker-compose.prod.yml` для конфигурации в Dokploy.
- Переменные окружения задаются через интерфейс Dokploy или `.env.production`.
- Весь стек (Apache, PHP, DB, Redis, Worker, Scheduler) разделен по сервисам.
- Внешний роутинг Dokploy должен быть направлен на сервис `laravel-apache-tcp` на порт 80.

## Стек

- Laravel 13+
- PHP 8.5 FPM (Alpine)
- Apache (Httpd) 2.4 (Alpine)
- PostgreSQL 18.2 (Alpine)
- Redis 8.6 (Alpine)
- Node.js 24 (Alpine)

## Примечания

- Dev-окружение использует bind mount проекта для удобства разработки.
- Взаимодействие Apache и PHP-FPM происходит через TCP внутри Docker-сети вместо Unix Domain Socket.
- В dev-режиме доступен `pgAdmin` для управления БД.
- Production-конфигурация ориентирована на создание неизменяемых (immutable) контейнеров.
