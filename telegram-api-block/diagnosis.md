# Диагностика: недоступность Telegram Bot API

## 1. Состояние контейнеров (TICKET-2, L2)

**Команда:**

```bash
$ docker ps -a
CONTAINER ID   IMAGE                             COMMAND                  CREATED          STATUS          PORTS                                                          NAMES
57ddc7329272   blog                              "docker-php-entrypoi…"   17 minutes ago   Up 10 minutes   9000/tcp                                                       laravel-worker
be0e65dc3662   blog                              "docker-php-entrypoi…"   17 minutes ago   Up 17 minutes   9000/tcp                                                       app
956e273f9a23   nginx:alpine                      "/docker-entrypoint.…"   17 minutes ago   Up 17 minutes   80/tcp                                                         blog-webserver
d42a1465f7b4   redis:alpine                      "docker-entrypoint.s…"   17 minutes ago   Up 17 minutes   6379/tcp, 0.0.0.0:6973->6973/tcp, :::6973->6973/tcp            redis
faf928c86ed8   jc21/nginx-proxy-manager:latest   "/init"                  3 months ago     Up 8 days       217.114.43.138:80-81->80-81/tcp, 217.114.43.138:443->443/tcp   nginxproxymanager
ba2a48c27aa4   mysql:8.0.34                      "docker-entrypoint.s…"   10 months ago    Up 8 days       33060/tcp, 127.0.0.1:33316->3306/tcp                           mysql-dev
```

**Наблюдения:**

- Все 6 контейнеров в состоянии `Up` — явных падений нет.
- `laravel-worker` поднят 10 минут при создании 17 минут назад — был перезапущен.
- `app` и `laravel-worker` собраны из одного образа `blog`

## 2. Логи воркера очереди (TICKET-2, L2)

**Команда:**

```bash
$ docker logs --tail 200 laravel-worker
2026-11-07 19:25:18 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:26:50 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:28:21 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:28:52 App\Jobs\SendTelegramMessage .............. 30 сек. FAIL
2026-11-07 19:51:57 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:53:29 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:55:00 App\Jobs\SendTelegramMessage ................... RUNNING
2026-11-07 19:55:30 App\Jobs\SendTelegramMessage .............. 30 сек. FAIL
```

**Наблюдения^**

- Задача `App\Jobs\SendTelegramMessage` циклически переходит в RUNNING и через ~30 секунд завершается FAIL.
- Между RUNNING и FAIL проходит ровно ~30 секунд (19:28:21 → 19:28:52; 19:55:00 → 19:55:30) — это признак таймаута соединения, а не ошибки в коде.
- Повторяемость паттерна (несколько циклов подряд) указывает на стабильно недоступный внешний ресурс, а не на случайный сбой.
- Других ошибок в логах воркера нет — падает именно SendTelegramMessage.

## 3. Лог приложения (TICKET-2, L2)

**Команда:**

```bash
$ docker exec -it app tail -n 300 storage/logs/laravel.log
[2026-11-07 19:28:52] local.ERROR: Telegram send failed permanently
{"error":"App\\Jobs\\SendTelegramMessage has timed out.",
 "data":{"username":"Иван Иванов","email":"ivanov@mail.ru","tg":"@ivanov",
         "message":"тест тест тест","company":null},
 "message":"📩 <b>Новая заявка с сайта</b> ..."}

[2026-11-07 19:55:30] local.ERROR: Telegram send failed permanently
{"error":"App\\Jobs\\SendTelegramMessage has timed out.",
 "data":{"username":"Иван Иванов","email":"ivanov@mail.ru","tg":"@ivanov",
         "message":"тест тест тест","company":null},
 "message":"📩 <b>Новая заявка с сайта</b> ..."}
```

**Наблюдения:**

Приложение фиксирует `Telegram send failed permanently` с причиной `SendTelegramMessage has timed out` — то есть job упал по таймауту, а не по ошибке логики.  
В записи присутствуют корректные данные формы (имя, email, telegram, сообщение) — значит, запрос от пользователя дошёл до бэкенда, валидация прошла, задача поставлена в очередь.  
Время в логе (19:28:52, 19:55:30) точно совпадает с временем FAIL в логах воркера (раздел 2) — это один и тот же инцидент, подтверждённый с двух сторон.  
Формирование текста сообщения для Telegram отрабатывает — в логе виден готовый HTML-шаблон. Проблема не в шаблоне, а в доставке.

**Промежуточный вывод:**

Приложение работает корректно: данные принимаются, задача ставится в очередь, текст сообщения формируется.  
Падение происходит на этапе HTTP-запроса к api.telegram.org — по таймауту.  
Подтверждает гипотезу из раздела 2: проблема сетевая, а не прикладная. Требуется проверка сетевой доступности

## 4. Сетевая доступность api.telegram.org (TICKET-3, L3)

### 4.1. Из контейнера app

**Команда:**

```bash
$ docker exec app curl -v --connect-timeout 10 https://api.telegram.org
*   Trying 149.154.166.110:443...
*   Trying [2001:67c:4e8:f004::9]:443...
* Immediate connect fail for 2001:67c:4e8:f004::9:
  Cannot assign requested address
* ipv4 connect timeout after 9997ms, move on!
* Failed to connect to api.telegram.org port 443 after 10000 ms:
  Timeout was reached
curl: (28) Failed to connect ... Timeout was reached
```

**Наблюдения:**

IPv4 (149.154.166.110:443): попытка соединения завершается таймаутом через ~10 секунд (`ipv4 connect timeout after 9997ms`).  
IPv6 (2001:67c:4e8:f004::9:443): мгновенная ошибка `Cannot assign requested address` — в контейнере нет назначенного IPv6-адреса для исходящих соединений.  
Итог: соединение с `api.telegram.org` не устанавливается ни по одному протоколу.

### 4.2. С хоста

**Команда:**

```bash
$ curl -v --connect-timeout 10 https://api.telegram.org
*   Trying 149.154.166.110:443...
*   Trying 2001:67c:4e8:f004::9:443...
* Immediate connect fail for 2001:67c:4e8:f004::9:
  Network is unreachable
*   Trying 2001:67c:4e8:f004::9:443...
* Immediate connect fail for 2001:67c:4e8:f004::9:
  Network is unreachable
...
* Connection timed out after 10000 milliseconds
curl: (28) Connection timed out after 10000 milliseconds
```

**Наблюдения:**

IPv4 (149.154.166.110:443): попытка соединения «висит» и завершается общим таймаутом `Connection timed out after 10000 milliseconds`.  
IPv6 (2001:67c:4e8:f004::9:443): мгновенная ошибка `Network is unreachable` — на хосте нет маршрута IPv6.  
`curl` многократно повторяет попытки IPv6 (по одной на каждый адрес из DNS-ответа), что дополнительно замедляет завершение команды.

### 4.3. Контрольная проверка

**Команда:**

```bash
$ curl -I --connect-timeout 5 https://www.google.com
HTTP/2 200
content-type: text/html; charset=ISO-8859-1
date: Sat, 07 Nov 2026 21:06:21 GMT
server: gws
```

## Вывод раздела 4

Исходящие соединения к api.telegram.org недоступны:

- с хоста — IPv4 таймаут, IPv6 Network is unreachable;
- из контейнера — IPv4 таймаут, IPv6 Cannot assign requested address.

При этом другие внешние ресурсы (google.com) доступны.
Это указывает на сетевое ограничение (блокировку/фильтрацию) на пути к Telegram, а не на ошибку приложения, конфигурации Docker или DNS.  
Требуется обходное решение — прокси-туннель для исходящих запросов к api.telegram.org.
