# [Infra] api.telegram.org недоступен из Docker-окружения

Приоритет: High
Компонент: Network / Docker
Автор: DevOps
Связан с: TICKET-2
Статус: In Progress → Resolved

## Описание

Проверил сетевую доступность api.telegram.org по тикету TICKET-2.

## Диагностика

- curl с хоста: IPv4 — таймаут, IPv6 — Network is unreachable.
- curl из контейнера app: IPv4 — connect timeout 10s, IPv6 — Cannot assign requested address.
- Вывод: исходящие соединения к `api.telegram.org` недоступны
  ни по IPv4, ни по IPv6 — ограничение на уровне сети/провайдера.

## Решение

- Поднят обходной прокси-туннель для Telegram Bot API.
- В HTTP-клиент Laravel добавлены proxy и `force_ip_resolve => 'v4'`.
- Перезапущен laravel-worker.

## Результат

Уведомления доставляются, в логах воркера — успешные DONE вместо FAIL. Тикет закрыт.

## Рекомендация

Настроить алерт на падение job'ов SendTelegramMessage, чтобы не ждать жалоб пользователей.
