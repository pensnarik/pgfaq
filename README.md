# PostgreSQL Frequently Asked Qestions

English version is [here](README-en.md)

Этот FAQ основан на вопросах, заданных в Telegram-канале https://t.me/pgsql и ответах на них.

## Блокировки

#### Как разрулить deadlock?

Главное, что нужно понять, это то, что deadlock всегда разруливается на стороне приложения и есть только 3 способа борьбы с ним:

1. Приложение, чья транзакция завершилась аварийно должно уметь обрабатывать эту ситуацию и повторять запрос.
2. Разные транзакции должны обновлять одни и те же данные в одном и том же порядке. В таком случае deadlock невозможен в принципе.
3. Приложение блокирует все строки, которые оно собирается обновить, непосредственно перед обновлением.

#### Что такое virtualxid?

virtualxid это "виртуальный" идентификатор транзакции, который состоит из pid бэкэнда и значения некоторого счётчика,
например "2/19". virtualxid назначается транзакции, которая не обновляла строки в базе данных а значит, ей не нужно
сохранять свой идентификатор в системных столбцах `xmin`, `xmax`. Делается это для того, чтобы 32 битный счётчик
"реальных" транзакций не закончился слишком быстро. Как только транзакция попытается обновить данные в базе данных
(или вызовет функцию `txid_current()`) ей будет назначен реальный `xid`.

## Работа с базой

#### Какие есть утилиты для работы с PostgreSQL?

1. Официальная консольная утилита [psql](https://www.postgresql.org/docs/current/static/app-psql.html)
2. [pgAdmin-III](https://www.pgadmin.org/docs/pgadmin3/1.22/)
3. [pgAdmin 4](https://www.pgadmin.org/)

#### Чем мониторить PostgreSQL?

##### Онлайн сервисы для мониторинга

1. [okmeter.io](https://okmeter.io/) - предоставляет довольно подробную статистику, но платный и требует создания специального пользователя в БД и запуска демона, который отвечает за отправку статистики на сервис.
2. [vividcortex.com](https://www.vividcortex.com/) - похожий по набору функций сервис, кроме PostgreSQL поддерживает Redis, MongoDB и MySQL.
3. [OpsDash](https://www.opsdash.com/) - ещё один онлайн серсис для мониторинга.
4. [pgwatch2](https://github.com/cybertec-postgresql/pgwatch2) - гибкое Open-Source решения для мониторинга PostgreSQL

##### Локальные утилиты

1. [PASH-Viewer](https://github.com/dbacvetkov/PASH-Viewer) - аналог ораклового Top Activity, написан на Java, см. [Мониторим активные сессии PostgreSQL 10, как в Oracle](https://habr.com/post/413411/)
2. [pg_activity](https://github.com/julmon/pg_activity) - консольное приложение (ncurses-based) для мониторинга текущих сессий в PostgreSQL
3. [pg_top](https://github.com/markwkm/pg_top) - 'top' для PostgreSQL

#### Как мне вставить в таблицу 1 миллиард записей? Быстро.

Вставка данных будет происходить быстрее, если на таблице отсутствуют индексы и внешние ключи. Для того, чтобы загрузить данные в таблицу из файла, можно воспользоваться командой [COPY](https://www.postgresql.org/docs/current/static/sql-copy.html). Если данные нужно сгенерировать "на лету", можно использовать функцию [generate_series](https://www.postgresql.org/docs/current/static/functions-srf.html). Например:

```sql
create table test(id bigint);
insert into test select n from generate_series(1, 1000000000) n;
```

Для того, чтобы следить за ходом выполнения операции (и не только, см. **Почему длинные транзакции это плохо?**), можно разбить `INSERT` на несколько или написать хранимую процедуру:

```sql
do $$
declare
  i integer;
begin
  for i in 1 .. 1000 loop
    insert into test select n from generate_series(1, 1000000) n;
    raise warning '% total rows inserted', i*1000000;
  end loop;
end;
$$ language plpgsql;
```

#### Почему длинные транзакции это плохо?

1. Препятствуют удалению ненужных строк автовакуумом
2. Опасность переполнения счётчика транзакций.
3. Блокировки.

Подробнее см. [Are Long Running Transactions Bad?](https://www.simononsoftware.com/are-long-running-transactions-bad/) и [Why avoid long transactions?](https://blog.dataegret.com/2018/12/why-avoid-long-transaction.html?m=1).

## Администрирование

#### Почему не удаляются сегменты WAL из pg_xlog?

1. Есть слот репликации, который препятствует удалению WAL файлов, проверить это можно следующим запросом:

```sql
SELECT slot_name,
       lpad((pg_control_checkpoint()).timeline_id::text, 8, '0') ||
       lpad(split_part(restart_lsn::text, '/', 1), 8, '0') ||
       lpad(substr(split_part(restart_lsn::text, '/', 2), 1, 2), 8, '0')
       AS wal_file
FROM pg_replication_slots;
```

2. Включено архивирование WAL, но команда `archive_command` не отрабатывает корректно (см. логи)
3. Количество файлов в `pg_xlog` ещё не превысило значение параметра `wal_keep_segments`
4. По какой-либо причине долгое время не происходит `chekpoint` (например, из-за ошибки)
