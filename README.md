# Laravel + Apache + PHP-FPM по TCP

Docker boilerplate для Laravel с раздельными контейнерами `httpd` и `php-fpm`.

Этот проект почти полностью повторяет вариант с `unix socket`, но здесь Apache подключается к PHP-FPM по TCP внутри Docker-сети:

- Apache проксирует PHP-запросы на `laravel-php-apache-tcp:9000`
- PHP-FPM слушает порт `9000` через `listen = 9000`
- общий Unix socket между контейнерами не используется

## Стек

- Laravel 13
- PHP 8.5 FPM (Alpine)
- Apache Httpd 2.4 (Alpine)
- PostgreSQL 18.2
- Redis 8.6
- Node 24 + Vite
- pgAdmin 4 для разработки
- отдельные контейнеры для queue worker и scheduler

## Архитектура

Основные dev-сервисы из [docker-compose.yml](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/docker-compose.yml):

- `laravel-php-apache-tcp` - контейнер приложения с PHP-FPM
- `laravel-apache-tcp` - фронтовый Apache-контейнер
- `laravel-node-apache-tcp` - Vite dev server
- `laravel-postgres-apache-tcp` - PostgreSQL
- `laravel-redis-apache-tcp` - Redis
- `laravel-queue-apache-tcp` - Laravel queue worker
- `laravel-scheduler-apache-tcp` - Laravel scheduler
- `laravel-pgadmin-apache-tcp` - pgAdmin

Схема запроса:

1. Браузер обращается к Apache на хостовом порту `HTTPD_PORT`
2. Apache отдает статику из `public/`
3. PHP-запросы уходят через `mod_proxy_fcgi` на `laravel-php-apache-tcp:9000`
4. Laravel подключается к PostgreSQL и Redis по именам контейнеров внутри bridge-сети Docker

Ключевое отличие от версии с Unix socket:

- вместо `unix:/path/to/php-fpm.sock|fcgi://...`
- используется `proxy:fcgi://laravel-php-apache-tcp:9000`

## Конфигурация

Базовые настройки для разработки находятся в [.env.example](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/.env.example).

Значения по умолчанию для dev:

- приложение: `http://localhost:8050`
- Vite HMR: `http://localhost:5173`
- pgAdmin: `http://localhost:8080`
- PostgreSQL: `localhost:5432`
- Redis: `localhost:6379`

Ключевые переменные окружения:

- `HTTPD_PORT=8050`
- `DB_HOST=laravel-postgres-apache-tcp`
- `DB_PORT=5432`
- `REDIS_HOST=laravel-redis-apache-tcp`
- `REDIS_PORT=6379`
- `XDEBUG_MODE=off`
- `XDEBUG_CLIENT_HOST=host.docker.internal`

## Быстрый старт

### Вариант 1. Полная инициализация через Makefile

```bash
cp .env.example .env
make setup
```

Что делает `make setup`:

- собирает образы
- поднимает контейнеры
- ждет готовности PostgreSQL и Redis
- ставит Composer- и NPM-зависимости
- генерирует `APP_KEY`
- запускает миграции
- исправляет права на `storage` и `bootstrap/cache`
- удаляет `public/.htaccess`, так как роутинг уже настроен напрямую в конфиге Apache

После запуска будут доступны:

- `http://localhost:8050`
- `http://localhost:8080` для pgAdmin

### Вариант 2. Ручной запуск

```bash
cp .env.example .env
docker compose -f docker-compose.yml build
docker compose -f docker-compose.yml up -d
docker compose -f docker-compose.yml exec laravel-php-apache-tcp composer install
docker compose -f docker-compose.yml exec laravel-php-apache-tcp php artisan key:generate
docker compose -f docker-compose.yml exec laravel-php-apache-tcp php artisan migrate
docker compose -f docker-compose.yml exec laravel-node-apache-tcp npm install
```

## Основные команды Makefile

Ключевые команды из [Makefile](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/Makefile):

```bash
make help
make setup
make up
make down
make build
make rebuild
make status
make logs
make validate
```

Полезные команды для сервисов:

```bash
make shell-php
make shell-httpd
make shell-postgres
make shell-redis
make logs-php
make logs-httpd
make logs-postgres
make logs-redis
```

Команды Laravel:

```bash
make artisan CMD="migrate"
make migrate
make rollback
make fresh
make tinker
make test-php
make test-coverage
```

Команды фронтенда:

```bash
make npm-install
make npm-dev
make npm-build
```

## Как настроено TCP-подключение

Конфиг Apache: [docker/httpd/conf/httpd.conf](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/docker/httpd/conf/httpd.conf)

```apache
<FilesMatch \.php$>
    SetHandler "proxy:fcgi://laravel-php-apache-tcp:9000"
</FilesMatch>
```

Конфиг PHP-FPM: [docker/php/www.conf](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/docker/php/www.conf)

```ini
listen = 9000
ping.path = /ping
ping.response = pong
```

Healthcheck PHP-FPM тоже идет по TCP через FastCGI ping, поэтому готовность контейнера проверяется не формально, а по реальному слушателю FPM на порту `9000`.

## Особенности разработки

- код приложения монтируется с хоста в PHP-, Apache- и Node-контейнеры
- Vite работает в отдельном Node-контейнере на порту `5173`
- в dev-образ PHP встроена поддержка Xdebug
- `clear_env = no` в FPM позволяет пробрасывать переменные контейнера внутрь PHP и Laravel
- сессии, кеш и очереди по умолчанию работают через Redis

## Production-режим

Конфигурация production разделена между файлами:

- [docker-compose.prod.yml](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/docker-compose.prod.yml)
- [docker-compose.prod.local.yml](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/docker-compose.prod.local.yml)
- [.env.production.example](/Users/vlavlamat/WorkSpace/php/laravel-labs/laravel-apache-tcp/.env.production.example)

Локальный запуск production-профиля:

```bash
cp .env.production.example .env.production
make up-prod
```

Что важно в production:

- PHP- и Apache-образы собираются как отдельные multi-stage production images
- frontend-ассеты собираются на этапе build
- PHP-контейнер при старте выполняет `php artisan migrate --force`
- queue worker и scheduler работают как отдельные long-running сервисы
- в local production режиме наружу публикуется только порт Apache

## Порты по умолчанию

- `8050` -> Apache
- `5173` -> Vite dev server
- `5432` -> PostgreSQL
- `6379` -> Redis
- `8080` -> pgAdmin
- `9000` -> внутренний TCP-порт PHP-FPM внутри Docker-сети

## Проверка работы

Проверить состояние контейнеров:

```bash
make status
```

Проверить HTTP-доступность сервисов:

```bash
make validate
```

Если что-то не работает, посмотреть логи:

```bash
make logs
make logs-php
make logs-httpd
```

## Примечание

- это не общий гайд по Laravel, а документация именно к этому Docker boilerplate
- основная цель проекта - показать связку Laravel + Apache + PHP-FPM через TCP вместо Unix socket
