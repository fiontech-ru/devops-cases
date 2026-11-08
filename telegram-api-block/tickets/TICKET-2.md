# [Feedback] Заявки не доходят до Telegram — подозрение на сеть

Приоритет: High
Компонент: Queue / SendTelegramMessage
Автор: Backend Developer
Связан с: TICKET-1
Статус: Open → эскалирован в инфраструктуру

## Описание

Проверил бэкенд по тикету TICKET-1.

## Что проверено

- Эндпоинт `POST /api/v1/feedback` отвечает `200 OK`.
- Задача `App\Jobs\SendTelegramMessage` встаёт в очередь.
- В логах воркера laravel-worker повторяющиеся FAIL с таймаутом ~30 секунд:

    ```bash
    2026-11-07 12:17:12 App\Jobs\SendTelegramMessage ... 30 сек. FAIL
    2026-11-07 12:20:47 App\Jobs\SendTelegramMessage ... 30 сек. FAIL
    ```

- Токен и chat_id в .env корректны, код job'а не менялся.

## Вывод

Код в порядке, job падает по таймауту соединения.
Похоже на сетевую недоступность api.telegram.org из контейнера.

Просьба к инфраструктуре: проверить исходящие соединения
к `api.telegram.org` с хоста и из контейнера.
