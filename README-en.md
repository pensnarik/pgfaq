# PostgreSQL Frequently Asked Qestions

English version is [here](README-en.md)

This FAQ is based on the questions and answers from [pgsql](https://t.me/pgsql) Telegram channgel.

## Locks

#### How do you handle deadlock?

The main point is deadlock should always be handled on the application side and there are only three ways we can hadle it:

1. Application should know about deadlocks and be able to deal with them. In case of deadlock application should repeat transaction cancelled by RDBMS.
2. Different transactions should update data in the same order. In this case deadlock is impossible.
3. Application should lock all the rows going to be updated before updating them.

## Working with database

#### What are the tools to work with PostgreSQL?

1. Official console tool [psql](https://www.postgresql.org/docs/current/static/app-psql.html)
2. [pgAdmin-III](https://www.pgadmin.org/docs/pgadmin3/1.22/)
3. [pgAdmin 4](https://www.pgadmin.org/)

#### How to monitor PostgreSQL?

##### Online tools

1. [okmeter.io](https://okmeter.io/) - provides detailed statistics but requires proprietary daemon to be run on your server.
2. [vividcortex.com](https://www.vividcortex.com/) - similar to the previous one but also supports Redis, MongoDB и MySQL.

##### Local tools

1. [PASH-Viewer](https://github.com/dbacvetkov/PASH-Viewer) - Oracle's Top Activity analog written in Java, for futher information see. [Monitoring PostgreSQL active sessions, the Oracle-way (in Russian)](https://habr.com/post/413411/)
2. [pg_activity](https://github.com/julmon/pg_activity) - ncurse-based application for active session monitoring.

#### How do very fast inserts with PostgreSQL.

Inserts are always faster on a table without indexes or foreign keys. In order to load data from file use [COPY](https://www.postgresql.org/docs/current/static/sql-copy.html) command. [generate_series](https://www.postgresql.org/docs/current/static/functions-srf.html) function can be used to generate data "on the fly":

```sql
create table test(id bigint);
insert into test select n from generate_series(1, 1000000000) n;
```

If you also want to monitor the progress of the operation (see also **Why long-running transactions are bad**), you can split `INSERT` into the smaller parts or write a stored procudure:

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

#### Why long-running transactions are bad?

1. They prevent vacuum from deleting old row versions.
2. Transaction number wraparound
3. Locks.

See [Are Long Running Transactions Bad?](https://www.simononsoftware.com/are-long-running-transactions-bad/) for defailed explanation.