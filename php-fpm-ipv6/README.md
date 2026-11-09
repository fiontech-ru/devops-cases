# Кейс: PHP-FPM слушает только IPv6 после обновления базового образа

После перехода на `php:8.2.35-fpm` (multi-stage build) Nginx перестал подключаться к PHP-FPM: `502 Bad Gateway`, в логах — `connect() failed (111: Connection refused)`.

## Проблема

Nginx (контейнер `blog-webserver`) подключается к `app:9000` по IPv4.  
PHP-FPM в контейнере `app` слушает порт 9000, но `/proc/net/tcp` (IPv4) пуст, а `/proc/net/tcp6` показывает `[::]:9000`.  
То есть FPM слушает **только IPv6**, и IPv4-соединения отвергаются.

## Диагностика

```bash
# Nginx: Connection refused
$ docker logs blog-webserver --tail 20
connect() failed (111: Connection refused) while connecting to upstream, upstream: "fastcgi://172.21.0.7:9000"

# FPM: конфиг говорит 9000, но IPv4-сокета нет
$ docker exec app php-fpm -tt 2>&1 | grep "listen ="
NOTICE: listen = 9000

$ docker exec app cat /proc/net/tcp | grep :2328
# (пусто — IPv4-сокета нет)

$ docker exec app cat /proc/net/tcp6 | grep :2328
00000000000000000000000000000000:2328 ...
# [::]:9000 — только IPv6
```

**Причина:** в docker.conf базового образа php:8.2.35-fpm стоит listen = 9000 без явного адреса.  
PHP-FPM в этом случае биндится на [::]:9000 (IPv6), а IPv4-сокет не создаётся.  
Это известное поведение PHP-FPM, подтверждённое баг-репортом.
[php-src #19987 ](https://github.com/php/php-src/issues/19987) (дубликат старых багов #77482 и #77501).

## Результат

```bash
$ docker exec app cat /proc/net/tcp | grep :2328
00000000:2328 ...
# 0.0.0.0:9000 — IPv4 работает

$ docker exec app cat /proc/net/tcp6 | grep :2328
# (пусто — IPv6 не используется, но это нормально)
```

Nginx подключается по IPv4, 502 исчез, сообщения доставляются.

## Что бы я сделал иначе

- Проверить `/proc/net/tcp` и `/proc/net/tcp6` сразу после пересборки — чтобы не ждать 502 от Nginx.
- Зафиксировать `listen = 0.0.0.0:9000` в Dockerfile как обязательную строку для multi-stage сборки, чтобы не зависеть от дефолтов базового образа.
