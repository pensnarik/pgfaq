# PostgreSQL Frequently Asked Qestions

Этот FAQ основан на вопросах, заданных в Telegram-канале https://t.me/pgsql и ответах на них.

## Блокировки

#### Как разрулить deadlock?

Главное, что нужно понять, это то, что deadlock всегда разруливается на стороне приложения и есть только 3 способа борьбы с ним:

1. Приложение, чья транзакция завершилась аварийно должна уметь обрабатывать эту ситуацию и повторить запрос.
2. Разные транзакции должны обновлять одни и те же данные в одном и том же порядке. В таком случае deadlock невозможен в принципе.
3. Приложение блокирует все строки, которые оно собирается обновить, непосредственно перед обновлением.

## Работа с базой

#### Какие есть утилиты для работы с PostgreSQL?

1. Официальная консольная утилита [psql](https://www.postgresql.org/docs/current/static/app-psql.html)
2. [pgAdmin-III](https://www.pgadmin.org/docs/pgadmin3/1.22/)
3. [pgAdmin 4](https://www.pgadmin.org/)

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

Пробробнее см. (Are Long Running Transactions Bad?)[https://www.simononsoftware.com/are-long-running-transactions-bad/]