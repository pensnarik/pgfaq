# PostgreSQL Frequently Asked Qestions

This FAQ is based on the questions and answers from [pgsql](https://t.me/pgsql) Telegram channgel.

## Locks

#### How do you handle deadlock?

The main point is deadlock should always be handled on the application side and there are only three ways we can hadle it:

1. Application should know about deadlocks and be able to deal with them. In case of deadlock application should repeat transaction cancelled by RDBMS.
2. Different transactions should update data in the same order. In this case deadlock is impossible.
3. Application should lock all the rows going to be updated before updating them.

#### What is virtualxid?

`virtualxid` is a virtual transaction counter that comprise backend id and backend local `xid` (for instance, "2/19").
`virtualxid` is allocated for transaction that haven't update any rows in the database therefore there is not need
to store transaction's `xid` in `xmin` and `xmax` system columns. This is to avoid wraparound of the system-wide
32-bit transaction ID. As soon as transaction updates any data in the database the real `xid` will be
allocated for this transaction (you can also call `txid_current()` function to force `xid` allocation).

## Working with database

#### What are the tools to work with PostgreSQL?

1. Official console tool [psql](https://www.postgresql.org/docs/current/static/app-psql.html)
2. [pgAdmin-III](https://www.pgadmin.org/docs/pgadmin3/1.22/)
3. [pgAdmin 4](https://www.pgadmin.org/)

#### How to monitor PostgreSQL?

##### Online tools

1. [okmeter.io](https://okmeter.io/) - provides detailed statistics but requires proprietary daemon to be run on your server.
2. [vividcortex.com](https://www.vividcortex.com/) - similar to the previous one but also supports Redis, MongoDB и MySQL.
3. [OpsDash](https://www.opsdash.com/) - another one PostgreSQL online monitoring service
4. [pgwatch2](https://github.com/cybertec-postgresql/pgwatch2) - flexible self-contained PostgreSQL metrics monitoring/dashboarding solution

##### Local tools

1. [PASH-Viewer](https://github.com/dbacvetkov/PASH-Viewer) - Oracle's Top Activity analog written in Java, for futher information see. [Monitoring PostgreSQL active sessions, the Oracle-way (in Russian)](https://habr.com/post/413411/)
2. [pg_activity](https://github.com/julmon/pg_activity) - ncurse-based application for active session monitoring.
3. [pg_top](https://github.com/markwkm/pg_top) - 'top' for PostgreSQL

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

See [Are Long Running Transactions Bad?](https://www.simononsoftware.com/are-long-running-transactions-bad/) and [Why avoid long transactions?](https://blog.dataegret.com/2018/12/why-avoid-long-transaction.html?m=1) for defailed explanation.

## Administration

#### Why PostgreSQL does not delete WAL files from pg_xlog?

There are several reasons why files might not been deleted:

1. There is a replication slot at a WAL location older than the WAL files, you can check it with this query:

```sql
SELECT slot_name,
       lpad((pg_control_checkpoint()).timeline_id::text, 8, '0') ||
       lpad(split_part(restart_lsn::text, '/', 1), 8, '0') ||
       lpad(substr(split_part(restart_lsn::text, '/', 2), 1, 2), 8, '0')
       AS wal_file
FROM pg_replication_slots;
```

2. WAL archiving is enabled and `archive_command` fails. Please, check PostgreSQL logs in this case.
3. `wal_keep_segments` is set higer than the number of WAL files in `pg_xlog`
4. There was not `checkpoint` for a long time (due to some error, etc)

#### How to determine which extension owns a procedure?

Analyze the output of:

```sql
\dx+ <extension-name>
```