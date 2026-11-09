# Кейс: оптимизация Docker-образа Laravel-приложения

Сборка Docker-образа занимала ~5 минут, размер — 1.44 GB. После комплексной оптимизации (multi-stage build, `.dockerignore`, обновление PHP и Node.js) cold build сократился до ~3 минут, размер — до 875 MB, warm build — до ~1 секунды.

## Проблема

- Cold build: **289 с**.
- Warm build (после изменения кода): **4.2 с**.
- Размер образа: **1.44 GB**.
- Базовый образ `php:8.2.23-fpm` содержал критические уязвимости
  (CVE-2026-109618/19, RCE, CVSS 9.2).
- Node.js 18 (EOL) устанавливался через `setup_18.x`, который
  не поддерживается на Debian 13 (Trixie).
- Пакет `software-properties-common` отсутствовал в Trixie,
  из-за чего сборка падала.

## Что сделано

### 1. Multi-stage build

Вынес компиляцию PHP-расширений в отдельный этап `builder`.  
В финальный образ копируются только готовые `.so`-файлы и конфиги (`COPY --from=builder`), без `build-essential`, `libssl-dev`, `pkg-config` и заголовочных файлов.

**Эффект:** размер образа уменьшился с 1.44 GB до 875 MB (−39%).

### 2. `.dockerignore`

Исключены из контекста сборки `vendor/`, `node_modules/`, `.git/`, `storage/logs/`. Контекст уменьшился с 61.64 MB до 872 KB.

### 3. Разделение слоёв и удаление `COPY` для Dev

Для Dev-окружения код монтируется через volume (`./:/var/www`), поэтому слой `COPY . /var/www` убран.  
При изменении кода пересобирается только слой с кодом (в Prod) или ничего (в Dev).

**Эффект:** warm build сократился с 4.2 с до ~1 с.

### 4. Обновление PHP

`php:8.2.23-fpm` → `php:8.2.35-fpm`. Устранены **CVE-2026-109618/19**.

### 5. Замена Node.js

`setup_18.x` (EOL) → `setup_24.x` (LTS, поддержка до 2028).
`npm` идёт в комплекте с `nodejs` из NodeSource.

### 6. Удаление `software-properties-common`

Пакет отсутствует в Debian 13 (Trixie) и не используется (нет `add-apt-repository`). Удалён из `apt-get install`.

### 7. Замена `pecl install redis` на сборку из исходников

PECL deprecated, `pecl install redis` падал с `Package "redis" does not have REST info xml available`.
Redis собирается из исходников GitHub через `docker-php-ext-install`.

### 8. Очистка APT в каждом `RUN`

`apt-get clean && rm -rf /var/lib/apt/lists/*` добавлен в те же
`RUN`-инструкции, где выполнялся `apt-get install`.

## Что пошло не так

При переходе на `php:8.2.35-fpm` FPM начал слушать только IPv6 (`[::]:9000`), из-за чего Nginx (контейнер `blog-webserver`) получал `Connection refused` и возвращал 502. 
Диагностика через `/proc/net/tcp6` выявила причину. Исправлено явным указанием `listen = 0.0.0.0:9000` в `docker.conf`.  
Подробнее — в [отдельном кейсе](../php-fpm-ipv6/README.md).

## Результат

| Метрика | До | После |
|---|---|---|
| Cold build (с нуля) | 289 с | **193.5 с** |
| Warm build (после изменения кода) | 4.2 с | **~1 с** |
| Размер образа | 1.44 GB | **875 MB** (−39%) |
| Контекст сборки | 61.64 MB | **872 KB** |
| PHP | 8.2.23 (CVE) | **8.2.35** (патчи) |
| Node.js | 18 (EOL) | **24 LTS** |

**Проверка расширений после multi-stage:**

```bash
$ docker run --rm blog php -m | grep -iE 'redis|gd|pdo_mysql|opcache'
gd
pdo_mysql
redis
Zend OPcache
```

Все расширения, включая opcache, загружены `— COPY --from=builder` корректно перенёс `.so` и `.ini`.

