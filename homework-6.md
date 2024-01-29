### **Домашнее задание 6**
#### *Работа с журналами*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

*Подключаемся на VM, устанавливаем PostgreSQL 15, обновляемся*
```bash
script@otus-pg-edu-vm:~$ sudo -i
root@otus-pg-edu-vm:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
*Настраиваем выполнение контрольной точки раз в 30 секунд с проверкой*
```bash
root@otus-pg-edu-vm:~ # egrep -i 'checkpoint_timeout' /etc/postgresql/15/main/postgresql.conf
#checkpoint_timeout = 5min              # range 30s-1d
root@otus-pg-edu-vm:~ # vim /etc/postgresql/15/main/postgresql.conf
checkpoint_timeout = 30s
root@otus-pg-edu-vm:~ # cat /etc/postgresql/15/main/postgresql.conf | grep checkpoint_timeout
checkpoint_timeout = 30s # range 30s-1d
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 main reload
```
*Генерируем нагрузочное тестирование в течении 10 минут c помощью pgbench*
```bash
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ pgbench -i postgres
postgres@otus-pg-edu-vm:~$ pgbench -c8 -P 60 -T 600 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 60.0 s, 708.7 tps, lat 11.282 ms stddev 11.082, 0 failed
progress: 120.0 s, 643.9 tps, lat 12.423 ms stddev 10.792, 0 failed
progress: 180.0 s, 553.1 tps, lat 14.460 ms stddev 19.508, 0 failed
progress: 240.0 s, 581.6 tps, lat 13.756 ms stddev 9.237, 0 failed
progress: 300.0 s, 569.4 tps, lat 14.047 ms stddev 11.939, 0 failed
progress: 360.0 s, 566.4 tps, lat 14.125 ms stddev 11.539, 0 failed
progress: 420.0 s, 555.6 tps, lat 14.395 ms stddev 9.797, 0 failed
progress: 480.0 s, 516.8 tps, lat 15.478 ms stddev 10.709, 0 failed
progress: 540.0 s, 506.7 tps, lat 15.790 ms stddev 10.787, 0 failed
progress: 600.0 s, 522.1 tps, lat 15.320 ms stddev 10.317, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 600 s
number of transactions actually processed: 343466
number of failed transactions: 0 (0.000%)
latency average = 13.974 ms
latency stddev = 11.944 ms
initial connection time = 19.636 ms
tps = 572.428186 (without initial connection time)
postgres@otus-pg-edu-vm:~$
```
*Проверяем объем журналов сгенерированных за это время*
```bash
root@otus-pg-edu-vm:~ # ls -lh /var/lib/postgresql/15/main/pg_wal/
total 65M
-rw------- 1 postgres postgres  16M янв 22 23:26 00000001000000000000001E
-rw------- 1 postgres postgres  16M янв 22 23:23 00000001000000000000001F
-rw------- 1 postgres postgres  16M янв 22 23:24 000000010000000000000020
-rw------- 1 postgres postgres  16M янв 22 23:24 000000010000000000000021
drwx------ 2 postgres postgres 4,0K янв 22 23:04 archive_status
root@otus-pg-edu-vm:~ #
```
**Создалось 4 архива по 16MB т.е. 64MB, в среднем на одну контрольную точку, которая создается каждые 30 секунд, в итоге выйдет ~3,2MB (64 / 20 = 3,2)**

*Проверяем данные статистики*
```bash
postgres=# select * from pg_stat_bgwriter \gx
-[ RECORD 1 ]---------+------------------------------
checkpoints_timed     | 30
checkpoints_req       | 0
checkpoint_write_time | 591742
checkpoint_sync_time  | 455
buffers_checkpoint    | 43472
buffers_clean         | 0
maxwritten_clean      | 0
buffers_backend       | 4058
buffers_backend_fsync | 0
buffers_alloc         | 4448
stats_reset           | 2024-01-22 23:10:11.786702+03

postgres=# show checkpoint_timeout;
 checkpoint_timeout
--------------------
 30s
(1 row)

postgres=#
```
**Все контрольные точки выполнялись каждые 30 секунд, по указанному расписанию**

*Сравниваем tps в синхронном/асинхронном режиме утилитой pgbench, пердаварительно нужно настроить два разных способа synchronous_commit*
- synchronous_commit = on
```bash
root@otus-pg-edu-vm:~ # cat /etc/postgresql/15/main/postgresql.conf  | grep synchronous_commit
#synchronous_commit = on                # synchronization level;
root@otus-pg-edu-vm:~ # vim /etc/postgresql/15/main/postgresql.conf
synchronous_commit = on
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 main reload
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ pgbench -c8 -P 20 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 642.9 tps, lat 12.427 ms stddev 8.453, 0 failed
progress: 40.0 s, 571.8 tps, lat 13.988 ms stddev 9.293, 0 failed
progress: 60.0 s, 586.5 tps, lat 13.640 ms stddev 9.596, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 36033
number of failed transactions: 0 (0.000%)
latency average = 13.319 ms
latency stddev = 9.130 ms
initial connection time = 18.134 ms
tps = 600.467416 (without initial connection time)
postgres@otus-pg-edu-vm:~$
```
- synchronous_commit = off
```bash
root@otus-pg-edu-vm:~ # vim /etc/postgresql/15/main/postgresql.conf
synchronous_commit = off
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 main reload
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ pgbench -c8 -P 20 -T 60 -U postgres postgres
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 20.0 s, 2464.0 tps, lat 3.242 ms stddev 1.497, 0 failed
progress: 40.0 s, 2162.0 tps, lat 3.699 ms stddev 1.768, 0 failed
progress: 60.0 s, 2181.9 tps, lat 3.665 ms stddev 1.772, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 8
number of threads: 1
maximum number of tries: 1
duration: 60 s
number of transactions actually processed: 136167
number of failed transactions: 0 (0.000%)
latency average = 3.523 ms
latency stddev = 1.690 ms
initial connection time = 17.611 ms
tps = 2269.647648 (without initial connection time)
postgres@otus-pg-edu-vm:~$
```
**Большая разница в tps между синхронным методом 600 tps и асинхронным методом 2269 tps. Так как в асинхронном режиме мы используем wal_writer_delay что увеличивает время отличка записи файл журнала wal на диск**

*Пункт 6, несколько этапов решения*
 - Создаем второй кластер с включенной контрольной суммой страниц*
```bash
postgres@otus-pg-edu-vm:~$ pg_createcluster 15 second -- --data-checksums
Creating new PostgreSQL cluster 15/second ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/second --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: ru_RU.UTF-8
  LC_NUMERIC:  ru_RU.UTF-8
  LC_TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/second ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Warning: systemd does not know about the new cluster yet. Operations like "service postgresql start" will not handle it. To fix, run:
  sudo systemctl daemon-reload
Ver Cluster Port Status Owner    Data directory                Log file
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
postgres@otus-pg-edu-vm:~$ pg_ctlcluster 15 second start
postgres@otus-pg-edu-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 online postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
postgres@otus-pg-edu-vm:~$
```
- Создаем таблицу и вносим данные, так же нужно узнать ее месторасположение на диске
```bash
postgres@otus-pg-edu-vm:~$ psql -p 5433
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table otus (id bigserial, text varchar(50));
CREATE TABLE
postgres=# insert into otus(text) select 'education' from generate_series(1,100);
INSERT 0 100
postgres=# select oid from pg_class where relname = 'otus';
  oid
-------
 16389
(1 row)

postgres=# select pg_relation_filepath('16389');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)

postgres=#
```
- Выключаем кластер
```bash
postgres@otus-pg-edu-vm:~$ pg_ctlcluster 15 second stop
postgres@otus-pg-edu-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
postgres@otus-pg-edu-vm:~$
```
- Изменяем пару байт в таблице
```bash
root@otus-pg-edu-vm:~ # apt install hexedit
root@otus-pg-edu-vm:~ # hexedit /var/lib/postgresql/15/second/base/5/16389
```
![image](https://github.com/gitsergeys/otus-pg-edu/assets/59079428/fa2f134b-6678-42f2-be40-4f7431d5175a)

- Запускаем кластер и делаем выборку данных из таблицы
```bash
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 second start
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ psql -p 5433
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | otus | table | postgres
(1 row)

postgres=# select * from otus limit 10;
 id |   text
----+-----------
  1 | education
  2 | education
  3 | education
  4 | education
  5 | education
  6 | education
  7 | education
  8 | education
  9 | education
 10 | education
(10 rows)

postgres=#
```
**Ошибок нет**

----
*Другой способ решения пункта 6*
 - Создаем второй кластер с включенной контрольной суммой страниц*
```bash
root@otus-pg-edu-vm:~ # pg_createcluster 15 second -- --data-checksums
Creating new PostgreSQL cluster 15/second ...
/usr/lib/postgresql/15/bin/initdb -D /var/lib/postgresql/15/second --auth-local peer --auth-host scram-sha-256 --no-instructions --data-checksums
The files belonging to this database system will be owned by user "postgres".
This user must also own the server process.

The database cluster will be initialized with this locale configuration:
  provider:    libc
  LC_COLLATE:  en_US.UTF-8
  LC_CTYPE:    en_US.UTF-8
  LC_MESSAGES: en_US.UTF-8
  LC_MONETARY: ru_RU.UTF-8
  LC_NUMERIC:  ru_RU.UTF-8
  LC_TIME:     ru_RU.UTF-8
The default database encoding has accordingly been set to "UTF8".
The default text search configuration will be set to "english".

Data page checksums are enabled.

fixing permissions on existing directory /var/lib/postgresql/15/second ... ok
creating subdirectories ... ok
selecting dynamic shared memory implementation ... posix
selecting default max_connections ... 100
selecting default shared_buffers ... 128MB
selecting default time zone ... Europe/Moscow
creating configuration files ... ok
running bootstrap script ... ok
performing post-bootstrap initialization ... ok
syncing data to disk ... ok
Ver Cluster Port Status Owner    Data directory                Log file
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 second start
root@otus-pg-edu-vm:~ # pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 online postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
root@otus-pg-edu-vm:~ #
```
- Создаем таблицу и вносим данные, так же нужно узнать ее месторасположение на диске
```bash
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ psql -p 5433
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table otus (id bigserial, text varchar(50));
CREATE TABLE
postgres=# insert into otus(text) select 'education' from generate_series(1,100);
INSERT 0 100
postgres=#  select oid from pg_class where relname = 'otus';
  oid
-------
 16389
(1 row)

postgres=# select pg_relation_filepath('16389');
 pg_relation_filepath
----------------------
 base/5/16389
(1 row)

postgres=# \q
postgres@otus-pg-edu-vm:~$
```
- Выключаем кластер
```bash
postgres@otus-pg-edu-vm:~$ pg_ctlcluster 15 second stop
postgres@otus-pg-edu-vm:~$ pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 down   postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
postgres@otus-pg-edu-vm:~$
```
- Изменяем пару байт в таблице, открываем файл в редакторе и удаляем несколько символов
```bash
vim /var/lib/postgresql/15/second/base/5/16389
:x!
```
![image](https://github.com/gitsergeys/otus-pg-edu/assets/59079428/40d8424f-ddd0-4c85-aa58-a81b4f7f90eb)

- Запускаем кластер и делаем выборку данных из таблицы
```bash
root@otus-pg-edu-vm:~ # pg_ctlcluster 15 second start
root@otus-pg-edu-vm:~ # pg_lsclusters
Ver Cluster Port Status Owner    Data directory                Log file
15  main    5432 online postgres /var/lib/postgresql/15/main   /var/log/postgresql/postgresql-15-main.log
15  second  5433 online postgres /var/lib/postgresql/15/second /var/log/postgresql/postgresql-15-second.log
root@otus-pg-edu-vm:~ # su postgres
postgres@otus-pg-edu-vm:/root$ cd ~
postgres@otus-pg-edu-vm:~$ psql -p 5433
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \dt
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | otus | table | postgres
(1 row)

postgres=# select * from otus limit 10;
 id | text
----+------
(0 rows)
```
**Ошибки снова нет, но и данных теперь вовсе нет**
