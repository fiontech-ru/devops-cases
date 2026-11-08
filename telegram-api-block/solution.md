# Решение: обход блокировки api.telegram.org

**Тикет:** TICKET-3 — [Infra] api.telegram.org недоступен из Docker-окружения
**Роль:** DevOps (L3)
**Статус:** Resolved
**Дата:** 2026-11-07

## Контекст

По результатам диагностики (`diagnosis.md`, разделы 1–4) установлено: исходящие соединения к `api.telegram.org` недоступны ни с хоста, ни из контейнера, ни по IPv4 (таймаут), ни по IPv6 (нет маршрута/адреса).  
При этом другие внешние ресурсы (`google.com`) доступны — проблема локализована в доступности Telegram, а не в сети сервера целиком.

Задача: обеспечить доставку сообщений из очереди Laravel в Telegram без изменения прикладной логики job'а.

## Архитектура решения

laravel-worker (контейнер)  
│ HTTP через proxy  
▼  
tgwsapiproxy (локальный прокси, SOCKS5/HTTP CONNECT)  
│ HTTPS  
▼  
Cloudflare Worker (long-bread-XXXX.X-email-name-X.workers.dev)  
│ HTTPS  
▼  
api.telegram.org

Локальный прокси принимает запросы от Laravel и заворачивает их в Cloudflare Worker, который, в свою очередь, обращается к Telegram Bot API.  
Прокси разрешает только трафик к `api.telegram.org`.

## Что сделал DevOps (L3)

1. **Развёрнут Cloudflare Worker**
    - Аккаунт Cloudflare, Worker создан через дашборд.
    - Домен: `long-bread-XXXX.X-email-name-X.workers.dev`
    - В Worker загружен скрипт `worker.js` из пакета `tgwsapiproxy`
      (автор — [winchanii](https://github.com/winchanii/TgWsApiProxy)).
    - Worker выступает внешним выходом туннеля к `api.telegram.org`.

2. **Подготовлен и запущен локальный прокси `tgwsapiproxy`**
    - Собран отдельный контейнер `tgwsapiproxy` на базе `python:3.12-alpine` (в проекте уже был `Dockerfile` для `app` — новый вынесен в подпапку `tgwsapiproxy/`, чтобы не смешивать образы).
    - Сервис добавлен в `docker-compose.yml`, подключён к сети `app-blog-network` — той же, что у app и `laravel-worker`.
    - Команда запуска (проверено через docker ps --no-trunc):

        ```bash
        tgwsapiproxy --worker-host long-bread-XXXX.X-email-name-X.workers.dev \
         --listen-host 0.0.0.0 \
         --listen-port 1080 \
         --http-port 8080
        ```

        `--listen-host 0.0.0.0` обязателен: по умолчанию прокси слушает только `127.0.0.1` внутри контейнера и недоступен другим сервисам.

    - Прокси доступен по имени сервиса tgwsapiproxy в сети `app-blog-network`.
    - Прокси разрешает только `api.telegram.org` (дефолтный `--allow`), то есть не является открытым релеем.
    - Параметры (--worker-host, порты) вынесены в .env и прокинуты через docker-compose.yml — Dockerfile остаётся универсальным.

3. **Проброс прокси в приложение**

- В `.env.example` добавлена переменная `TELEGRAM_PROXY_URL` (значение по умолчанию: `http://tgwsapiproxy:8080`).
- В .env добавлена `TELEGRAM_PROXY_URL=http://tgwsapiproxy:8080`. Так как .env примонтирован в контейнеры, Laravel читает её через Dotenv — явный проброс через environment не требуется.
- Адрес прокси — по имени сервиса (`tgwsapiproxy:8080`), а не 127.0.0.1, так как контейнеры находятся в одной Docker-сети.

4. **Проверка сетевой связности**

    Тест из контейнера app (через HTTP CONNECT прокси):

    ```bash
    $ TOKEN=$(grep '^TELEGRAM_BOT_TOKEN=' .env | cut -d= -f2-)
    $ docker exec app curl -sS --proxy http://tgwsapiproxy:8080 "https://api.telegram.org/bot${TOKEN}/getMe"
    ```

    **Результат:**

    ```json
    {"ok":true,"result":{"id":<BOT_ID>,"is_bot":true,"first_name":"pdbot",
    "username":"PdStudioBot",...}}
    ```

**Вывод:**

Цепочка app → tgwsapiproxy → Cloudflare Worker → api.telegram.org работает, токен валиден.  
Инфраструктурная часть готова к использованию из Laravel.

Известное поведение (транзиентные ошибки):  
В одном из запусков возникла ошибка `curl: (35) OpenSSL SSL_connect: SSL_ERROR_SYSCALL in connection to api.telegram.org:443`.  
Повторный запуск той же команды прошёл успешно. Причина — холодный старт изолятов Cloudflare Worker: первое TLS-рукопожатие может оборваться, пока изолят инициализируется.
Пакет tgwsapiproxy уже делает до 4 повторных попыток рукопожатия с растущим бэкоффом, но curl при обрыве на уровне TLS не всегда успевает получить ответ от прокси.

Для приложения это компенсируется на стороне Laravel: HTTP-клиент повторяет запрос при ConnectionException (см. раздел «Что сделал разработчик»).  
Для инфраструктуры дополнительных действий не требуется — ошибка непостоянная и не связана с конфигурацией прокси или Worker'а.

### 5. Эксплуатация

**Автозапуск.**
Контейнер `tgwsapiproxy` настроен с `restart: unless-stopped` — поднимается автоматически после перезагрузки хоста или падения процесса.  
Cloudflare Worker работает на стороне Cloudflare и не требует обслуживания на сервере.

**Устойчивость к холодному старту.**
При первом запросе после простоя изолят Cloudflare Worker инициализируется, и TLS-рукопожатие может оборваться (`SSL_ERROR_SYSCALL`).  
Пакет `tgwsapiproxy` компенсирует это до 4 повторными попытками хендшейка с растущим бэкоффом.
На стороне Laravel добавлен `retry` для `ConnectionException` — транзиентные обрывы не приводят к падению job'а.

**Безопасность.**
Прокси ограничен api.telegram.org (--allow по умолчанию), не является открытым релеем. Параметры вынесены в .env, Dockerfile универсален.

**Работа без прокси.**
Если TELEGRAM_PROXY_URL пуст, config() возвращает null, клиент идёт напрямую — одна кодовая база для окружений с блокировкой и без.

**Диагностика.**

- Состояние контейнера: `docker ps --no-trunc --filter "name=tgwsapiproxy"`.
- Логи прокси: `docker logs --tail 100 tgwsapiproxy`.
- Проверка связности: `getMe` через прокси из контейнера `app` — команда и ожидаемый результат в разделе 4.

## Что сделал разработчик (L2)

### 1. Правка `AbstractTelegramService`

- Добавлен метод `http()` — преднастроенный клиент для запросов к Bot API:

    ```php
    protected function http(): PendingRequest
    {
        $options = ['force_ip_resolve' => 'v4'];

        if ($proxy = config('services.telegram.proxy_url')) {
            $options['proxy'] = $proxy;
        }

        return Http::withOptions($options)->timeout(10);
    }
    ```

- proxy — адрес tgwsapiproxy из config('services.telegram.proxy_url').
- force_ip_resolve => 'v4' — в окружении нет рабочего IPv6.
- Проверка if ($proxy = ...) обеспечивает работу без прокси: если TELEGRAM_PROXY_URL не задан, клиент идёт напрямую.

В config/services.php добавлено 'proxy_url' => env('TELEGRAM_PROXY_URL').

### 2. Правка sendMessage в V1 и V2

Http::timeout(10) заменён на $this->http(), retry получил callback,
чтобы повторять только сетевые ошибки:

```php
$response = $this->http()
    ->retry(3, 100, function ($exception) {
        return $exception instanceof \Illuminate\Http\Client\ConnectionException;
    })
    ->post("{$this->apiUrl}/sendMessage", $payload);
```

Без callback retry повторял бы и 4xx (неверный токен/chat_id), что бессмысленно. Callback ограничивает повторы ConnectionException (таймауты, TLS-обрывы SSL_ERROR_SYSCALL).
Приложение работает с прокси и без — переключение через TELEGRAM_PROXY_URL, без изменения кода. Ретраи ограничены сетевыми ошибками, правка вынесена в базовый класс и общая для V1 и V2.

## Результат

Уведомления из формы обратной связи доставляются в Telegram.
Job SendTelegramMessage больше не падает по таймауту.
В логах воркера — DONE вместо FAIL.

Инфраструктурная задача решена: DevOps развернул прокси-туннель, разработчик подключил его в HTTP-клиенте и добавил ретраи на сетевые ошибки.
Бизнес-логика (SendTelegramMessage) не менялась — правки вынесены в базовый класс.

## Что осталось / рекомендации

- Настроить алерт на падение job'ов SendTelegramMessage, чтобы не ждать жалоб пользователей.
- Задокументировать в runbook: адрес Worker, порт прокси, как проверить связность.
- Рассмотреть резервный канал доставки (email) на случай недоступности Telegram.
