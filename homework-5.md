### **Домашнее задание 5**
#### *Настройка autovacuum с учетом особеностей производительности*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

*Подключаемся на VM и устанавливаем PostgreSQL 15, обновляемся*
```bash
script@otus-pg-edu-vm:~$ sudo -i
root@otus-pg-edu-vm:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
*Проверяем кластер*
```bash
root@otus-pg-edu-vm:~ # pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
root@otus-pg-edu-vm:~ #
```
*Создаем БД для тестов и выполняем pgbench -i postgres*
```bash
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database otus;
CREATE DATABASE
postgres=#\q
postgres@otus-pg-edu-vm:~$ pgbench -i postgres
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.04 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.27 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.15 s, vacuum 0.03 s, primary keys 0.06 s).
postgres@otus-pg-edu-vm:~$
```
*Запускаем pgbench -c8 -P 6 -T 60 -U postgres postgres*
```bash
postgres@otus-pg-edu-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1187.5 tps, lat 6.711 ms stddev 5.940, 0 failed
progress: 12.0 s, 1094.2 tps, lat 7.312 ms stddev 5.454, 0 failed
progress: 18.0 s, 1112.7 tps, lat 7.192 ms stddev 5.151, 0 failed
progress: 24.0 s, 1092.0 tps, lat 7.323 ms stddev 5.715, 0 failed
progress: 30.0 s, 1099.1 tps, lat 7.278 ms stddev 5.827, 0 failed
progress: 36.0 s, 935.7 tps, lat 8.554 ms stddev 19.638, 0 failed
progress: 42.0 s, 1126.0 tps, lat 7.101 ms stddev 5.122, 0 failed
progress: 48.0 s, 1112.6 tps, lat 7.186 ms stddev 5.259, 0 failed
progress: 54.0 s, 1003.2 tps, lat 7.980 ms stddev 5.698, 0 failed
progress: 60.0 s, 1032.0 tps, lat 7.749 ms stddev 4.740, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 64778
number of failed transactions: 0 (0.000%)
latency average = 7.409 ms
latency stddev = 7.798 ms
initial connection time = 14.153 ms
tps = 1079.554652 (without initial connection time)
postgres@otus-pg-edu-vm:~$
```
*Применяем параметры настройки PostgreSQL из прикрепленного к материалам занятия файла*
```bash
root@otus-pg-edu-vm:~ # cat /etc/postgresql/15/main/postgresql.conf | egrep -i 'max_connections|shared_buffers|effective_cache_size|maintenance_work_mem|checkpoint_completion_target|wal_buffers|default_statistics_target|random_page_cost|effective_io_concurrency|work_mem|min_wal_size|max_wal_size'
max_connections = 40
shared_buffers = 1GB
work_mem = 6553kB
maintenance_work_mem = 512MB
effective_io_concurrency = 2
wal_buffers = 16MB
checkpoint_completion_target = 0.9
max_wal_size = 16GB
min_wal_size = 4GB
random_page_cost = 4 
effective_cache_size = 3GB
default_statistics_target = 500
root@otus-pg-edu-vm:~ #
```
*Снова запускаем pgbench -c8 -P 6 -T 60 -U postgres postgres*
```bash
postgres@otus-pg-edu-vm:~$ pgbench -c8 -P 6 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 6.0 s, 1012.5 tps, lat 7.875 ms stddev 5.966, 0 failed
progress: 12.0 s, 896.7 tps, lat 8.917 ms stddev 6.322, 0 failed
progress: 18.0 s, 960.5 tps, lat 8.333 ms stddev 5.815, 0 failed
progress: 24.0 s, 1055.5 tps, lat 7.576 ms stddev 5.061, 0 failed
progress: 30.0 s, 1068.2 tps, lat 7.489 ms stddev 5.103, 0 failed
progress: 36.0 s, 962.7 tps, lat 8.311 ms stddev 5.762, 0 failed
progress: 42.0 s, 914.0 tps, lat 8.752 ms stddev 6.025, 0 failed
progress: 48.0 s, 883.9 tps, lat 9.052 ms stddev 5.879, 0 failed
progress: 54.0 s, 871.5 tps, lat 9.159 ms stddev 5.646, 0 failed
progress: 60.0 s, 814.5 tps, lat 9.834 ms stddev 6.161, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 56647
number of failed transactions: 0 (0.000%)
latency average = 8.473 ms
latency stddev = 5.809 ms
initial connection time = 13.784 ms
tps = 943.961479 (without initial connection time)
postgres@otus-pg-edu-vm:~$
```

**Почти ни чего не изменилось / Изменилось кол-во транзакций в секунду tps и время подкючения, не могу сказать почему...**

*Создаем таблицу с текстовым полем и заполняем данными в размере 1млн строк*
```bash
root@ats-test-2:/ # su postgres
bash-4.2$ psql
psql (15.4)
Type "help" for help.

postgres=# create table otus (id bigserial, text varchar(50));
CREATE TABLE
postgres=# insert into otus(text) select 'education' from generate_series(1,1000000);
INSERT 0 1000000
postgres=#
```
*Смотрим размер файла с таблицей*
```bash
postgres=# select pg_size_pretty(pg_table_size('otus'));
 pg_size_pretty
----------------
 50 MB
(1 row)

postgres=#
```
*Обновляем таблицу с текстовым полем 5 раз, изменив текст*
```bash
postgres=# update otus set text = 'education_sample';
UPDATE 1000000
postgres=# update otus set text = 'education_sample';
UPDATE 1000000
postgres=# update otus set text = 'education_sample';
UPDATE 1000000
postgres=# update otus set text = 'education_sample';
UPDATE 1000000
postgres=# update otus set text = 'education_sample';
UPDATE 1000000
postgres=#
```
*Смотрим количество мертвых строчек в таблице и когда последний раз приходил автовакуум*
```bash
postgres=# select relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum from pg_stat_user_tables where relname = 'otus';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 otus    |     995758 |          0 |      0 | 2024-01-18 18:53:31.707974+03
(1 row)

postgres=#
```
*Снова обновляем таблицу с текстовым полем 5 раз, изменив текст и проверяем ее размер*
```bash
postgres=# update otus set text = 'education_sample_2';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_2';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_2';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_2';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_2';
UPDATE 1000000
postgres=# select pg_size_pretty(pg_table_size('otus'));
 pg_size_pretty
----------------
 345 MB
(1 row)

postgres=#
```
*Смотрим количество мертвых строчек в таблице и когда последний раз приходил автовакуум*
```bash
postgres=# select relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum from pg_stat_user_tables where relname = 'otus';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 otus    |     994846 |          0 |      0 | 2024-01-18 18:59:31.663425+03
(1 row)

postgres=#
```
*Проеряем на всякий случай настройки*
```bash
postgres=# select name, setting from pg_settings where name like 'autovacuum%';
                 name                  |  setting
---------------------------------------+-----------
 autovacuum                            | on
 autovacuum_analyze_scale_factor       | 0.1
 autovacuum_analyze_threshold          | 50
 autovacuum_freeze_max_age             | 200000000
 autovacuum_max_workers                | 3
 autovacuum_multixact_freeze_max_age   | 400000000
 autovacuum_naptime                    | 60
 autovacuum_vacuum_cost_delay          | 2
 autovacuum_vacuum_cost_limit          | -1
 autovacuum_vacuum_insert_scale_factor | 0.2
 autovacuum_vacuum_insert_threshold    | 1000
 autovacuum_vacuum_scale_factor        | 0.2
 autovacuum_vacuum_threshold           | 50
 autovacuum_work_mem                   | -1
(14 rows)

postgres=#
```
*Отключаем Автовакуум на этой таблице otus*
```bash
postgres=# alter table otus set (autovacuum_enabled = off);
ALTER TABLE
postgres=#
```
*10 раз обновляем инфо в таблице и смотрим размеры таблицы*
```bash
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# update otus set text = 'education_sample_3';
UPDATE 1000000
postgres=# select pg_size_pretty(pg_table_size('otus'));
 pg_size_pretty
----------------
 632 MB
(1 row)

postgres=#
```

**Каждый update создает мертвую строчку как и insert/upddat/delete - таблица как и БД разрастется в целом**

**Нужно включать на такие случаи параметр autovacuum, кроме full чтобы замещать пустое место новыми данными**

*Включаем и наслаждаемся*
```bash
postgres=# alter table otus set (autovacuum_enabled = on);
ALTER TABLE
postgres=# select relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum from pg_stat_user_tables where relname = 'otus';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum
---------+------------+------------+--------+-------------------------------
 otus    |    1001729 |          0 |      0 | 2024-01-18 19:44:45.338124+03
(1 row)

postgres=#
```
*Делаем vacuum full и смотрим размеры*
```bash
postgres=# vacuum full;
VACUUM
postgres=# select pg_size_pretty(pg_table_size('otus'));
 pg_size_pretty
----------------
 57 MB
(1 row)

postgres=#
```
*Вот и все =)*
****************
#### **Дополнительное задание со звездочкой**
-------------------------------------------------------
*Процедуру тестировал в Navicate все отработало, в psql которым никогда не пользуюсь не тестировал*
```sql
create or replace functions update_text_values() 
returns void as $$
declare
    counter int := 1;
begin
    while counter <= 10 loop
        raise notice 'Step %', counter;
        update otus 
        set text = text || ' otus_education'
        counter := counter + 1;
    end loop;
end;
$$ language plpgsql;
```
*Вызывать через команду*
```bash
select update_text_values();
```
