# DevOps Cases

Практические разборы инфраструктурных инцидентов: диагностика, решения,
что можно было сделать иначе. Каждый кейс — отдельная папка с README,
подробной диагностикой, решением и тикетами (по ролям L1/L2/L3).

## Кейсы

### [Обход блокировки api.telegram.org](./devops-cases/telegram-api-block/README.md)

Форма обратной связи возвращала `200 OK`, но уведомления в Telegram
не приходили. Диагностика выявила сетевую недоступность
`api.telegram.org` (ни IPv4, ни IPv6). Решение — локальный прокси
`tgwsapiproxy` через Cloudflare Worker.

- [Обзор кейса](./telegram-api-block/README.md)
- [Диагностика](./telegram-api-block/diagnosis.md)
- [Решение](./telegram-api-block/solution.md)
- Тикеты:
    - [TICKET-1 — обнаружение (L1)](./telegram-api-block/tickets/TICKET-1.md)
    - [TICKET-2 — разбор разработчиком (L2)](./telegram-api-block/tickets/TICKET-2.md)
    - [TICKET-3 — инфраструктура (L3)](./telegram-api-block/tickets/TICKET-3.md)

## Структура репозитория

```text
devops-cases/
├── README.md
└── cases/
    └── telegram-api-block/
        ├── README.md          # краткий обзор кейса
        ├── diagnosis.md       # диагностика: команды, выводы, наблюдения
        ├── solution.md        # решение: архитектура, что сделал DevOps/Developer
        └── tickets/
            ├── TICKET-1.md    # L1: обнаружение
            ├── TICKET-2.md    # L2: первичный разбор
            └── TICKET-3.md    # L3: инфраструктура
```

## Формат кейса

Каждый кейс следует структуре:

1. **README.md** — краткий обзор: проблема, как обнаружили, разбор по ролям,
   результат.
2. **diagnosis.md** — пошаговая диагностика с командами и выводами.
3. **solution.md** — архитектура решения, что сделал DevOps (L3) и разработчик (L2),
   эксплуатация, рекомендации.
4. **tickets/** — тикеты по линиям поддержки, как они выглядели бы в трекере.

## Автор

Maintained by [@drfoxg](https://github.com/drfoxg) · [fiontech.ru](https://fiontech.ru)
