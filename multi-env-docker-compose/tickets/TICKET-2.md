# TICKET-2: Добавить соединение mysql_migrator в config/database.php

**Открыл:** DevOps  
**Исполнитель:** Dev (Laravel)  
**Статус:** Done

## Контекст

Для prod миграции должны идти под `blog_prod_migrator` (`ALL PRIVILEGES`), а приложение — под `blog_prod_user` (только `SELECT, INSERT, UPDATE, DELETE`). Это снижает риск: если в приложении появится SQL-инъекция, злоумышленник не сможет удалить таблицы.

Нужно отдельное соединение `mysql_migrator` в `config/database.php`, которое берёт `username`/`password` из `DB_MIGRATOR_*`.

## Что сделать (Dev)

- В `config/database.php` добавить `'mysql_migrator'` — копия `'mysql'`, но `username` => `env('DB_MIGRATOR_USERNAME')`, `password` => `env('DB_MIGRATOR_PASSWORD')`.
- В `.env.example` добавить `DB_MIGRATOR_USERNAME=`, `DB_MIGRATOR_PASSWORD=`.
- В `.env.prod` (на сервере) добавить реальные значения (не коммитить).

## Критерии приёмки

- [ ] `php artisan migrate --database=mysql_migrator` работает в **prod** (после добавления `DB_MIGRATOR_*` в `.env.prod`)
- [ ] приложение (без `--database`) работает под `blog_prod_user` (runtime)
- [ ] в staging/dev миграции идут под `blog_staging_user`/`blog_dev_user` (без `--database`), это ожидаемо

## Результат

Соединение добавлено, миграции в prod идут под `blog_prod_migrator`, приложение — под `blog_prod_user`.
