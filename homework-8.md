### **Домашнее задание 8**
#### *Настройка PostgreSQL*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

Нагрузочное тестирование и тюнинг PostgreSQL
Цель:
- сделать нагрузочное тестирование PostgreSQL
- настроить параметры PostgreSQL для достижения максимальной производительности
-------------------------------------------------------
*Виртуальная машина и PostgreSQL развернуты аналогичным способом, уже пердустановлены*

*Необходимо настроить кластер postgresql 15 на максимальную производительность*

*Проводим измерение нагрузки через pgbench*
- 50 клиентов `--client=50`
- новое подключение для каждой транзакции `--connect`
- 5 параллельных потоков `--jobs=5`
- выводим прогрес каждые 30 секунд `--progress=30`
- время на тест 3 минуты `--time=180`
- БД otus_db_bench

```bash
root@otus-edu-pg-vm-1:~ # su postgres
postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \l
                                                 List of databases
   Name    |  Owner   | Encoding |   Collate   |    Ctype    | ICU Locale | Locale Provider |   Access privileges
-----------+----------+----------+-------------+-------------+------------+-----------------+-----------------------
 postgres  | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            |
 template0 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
 template1 | postgres | UTF8     | en_US.UTF-8 | en_US.UTF-8 |            | libc            | =c/postgres          +
           |          |          |             |             |            |                 | postgres=CTc/postgres
(3 rows)

postgres=# create database otus_db_bench;
CREATE DATABASE
postgres=# \q
postgres@otus-edu-pg-vm-1:/root$ pgbench -i otus_db_bench
dropping old tables...
NOTICE:  table "pgbench_accounts" does not exist, skipping
NOTICE:  table "pgbench_branches" does not exist, skipping
NOTICE:  table "pgbench_history" does not exist, skipping
NOTICE:  table "pgbench_tellers" does not exist, skipping
creating tables...
generating data (client-side)...
100000 of 100000 tuples (100%) done (elapsed 0.12 s, remaining 0.00 s)
vacuuming...
creating primary keys...
done in 0.48 s (drop tables 0.00 s, create tables 0.02 s, client-side generate 0.29 s, vacuum 0.05 s, primary keys 0.11 s).
postgres@otus-edu-pg-vm-1:/root$ pgbench --client=50 --connect --jobs=5 --progress=30 --time=180 otus_db_bench
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 147.8 tps, lat 324.998 ms stddev 326.958, 0 failed
progress: 60.0 s, 156.6 tps, lat 313.187 ms stddev 335.220, 0 failed
progress: 90.0 s, 158.9 tps, lat 306.876 ms stddev 314.953, 0 failed
progress: 120.0 s, 157.3 tps, lat 305.918 ms stddev 318.041, 0 failed
progress: 150.0 s, 157.6 tps, lat 312.181 ms stddev 336.326, 0 failed
progress: 180.0 s, 166.4 tps, lat 293.699 ms stddev 295.026, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 5
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 28387
number of failed transactions: 0 (0.000%)
latency average = 309.326 ms
latency stddev = 321.247 ms
average connection time = 7.793 ms
tps = 157.557910 (including reconnection times)
postgres@otus-edu-pg-vm-1:/root$
```
*Генерируем конфигурацию для максимальной производительности и создаем файл (например через ресурс https://pgconfigurator.cybertec.at/)*

```bash
# Connectivity
max_connections = 100
superuser_reserved_connections = 3

# Memory Settings
shared_buffers = '1024 MB'
work_mem = '32 MB'
maintenance_work_mem = '320 MB'
huge_pages = off
effective_cache_size = '3 GB'
effective_io_concurrency = 100 # concurrent IO only really activated if OS supports posix_fadvise function
random_page_cost = 1.25 # speed of random disk access relative to sequential access (1.0)

# Monitoring
shared_preload_libraries = 'pg_stat_statements' # per statement resource usage stats
track_io_timing=on # measure exact block IO times
track_functions=pl # track execution times of pl-language procedures if any

# Replication
wal_level = replica # consider using at least 'replica'
max_wal_senders = 0
synchronous_commit = off

# Checkpointing:
checkpoint_timeout = '15 min'
checkpoint_completion_target = 0.9
max_wal_size = '1024 MB'
min_wal_size = '512 MB'

# WAL writing
wal_compression = on
wal_buffers = -1 # auto-tuned by Postgres till maximum of segment size (16MB by default)
wal_writer_delay = 200ms
wal_writer_flush_after = 1MB

# Background writer
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
bgwriter_flush_after = 0

# Parallel queries:
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_maintenance_workers = 2
max_parallel_workers = 4
parallel_leader_participation = on

# Advanced features
enable_partitionwise_join = on
enable_partitionwise_aggregate = on
jit = on
max_slot_wal_keep_size = '1000 MB'
track_wal_io_timing = on
maintenance_io_concurrency = 100
wal_recycle = off

postgres@otus-edu-pg-vm-1:/root$ vim /etc/postgresql/15/main/conf.d/tune.conf
postgres@otus-edu-pg-vm-1:/root$ exit
root@otus-edu-pg-vm-1:~ # systemctl restart postgresql@15-main
root@otus-edu-pg-vm-1:~ # systemctl status postgresql@15-main.service
● postgresql@15-main.service - PostgreSQL Cluster 15-main
     Loaded: loaded (/lib/systemd/system/postgresql@.service; enabled-runtime; vendor preset: enabled)
     Active: active (running) since Sun 2024-04-14 11:28:07 MSK; 12s ago
    Process: 31165 ExecStart=/usr/bin/pg_ctlcluster --skip-systemctl-redirect 15-main start (code=exited, status=0/SUCCESS)
   Main PID: 31170 (postgres)
      Tasks: 6 (limit: 4598)
     Memory: 50.0M
        CPU: 124ms
     CGroup: /system.slice/system-postgresql.slice/postgresql@15-main.service
             ├─31170 /usr/lib/postgresql/15/bin/postgres -D /var/lib/postgresql/15/main -c config_file=/etc/postgresql/15/main/postgresql.conf
             ├─31171 "postgres: 15/main: checkpointer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─31172 "postgres: 15/main: background writer " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" >
             ├─31174 "postgres: 15/main: walwriter " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "">
             ├─31175 "postgres: 15/main: autovacuum launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">
             └─31176 "postgres: 15/main: logical replication launcher " "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" "" ">

апр 14 11:28:05 otus-edu-pg-vm-1 systemd[1]: Starting PostgreSQL Cluster 15-main...
апр 14 11:28:07 otus-edu-pg-vm-1 systemd[1]: Started PostgreSQL Cluster 15-main.
lines 1-18/18 (END)
root@otus-edu-pg-vm-1:~ #
```

*Проводим нагрузку с новыми параметрами*
```bash
root@otus-edu-pg-vm-1:~ # su postgres
postgres@otus-edu-pg-vm-1:/root$ pgbench --client=50 --connect --jobs=5 --progress=30 --time=180 otus_db_bench
pgbench (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
starting vacuum...end.
progress: 30.0 s, 148.2 tps, lat 327.397 ms stddev 300.709, 0 failed
progress: 60.0 s, 157.2 tps, lat 311.750 ms stddev 289.951, 0 failed
progress: 90.0 s, 157.5 tps, lat 312.717 ms stddev 288.962, 0 failed
progress: 120.0 s, 137.5 tps, lat 355.523 ms stddev 392.449, 0 failed
progress: 150.0 s, 133.4 tps, lat 362.461 ms stddev 356.605, 0 failed
progress: 180.0 s, 155.0 tps, lat 315.811 ms stddev 313.385, 0 failed
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 1
query mode: simple
number of clients: 50
number of threads: 5
maximum number of tries: 1
duration: 180 s
number of transactions actually processed: 26713
number of failed transactions: 0 (0.000%)
latency average = 329.829 ms
latency stddev = 324.405 ms
average connection time = 7.234 ms
tps = 248.220490 (including reconnection times)
postgres@otus-edu-pg-vm-1:/root$
```
Получили + 91 tps от стандартной конфигурации, при таких настройках

*Важные параметры*

shared_buffers = '1024 MB'
Отвечает за выделение памяти для кэширования данных, считываемых из диска. Это позволяет уменьшить количество физических операций ввода-вывода (I/O), так как данные, к которым обращаются часто, будут находиться в оперативной памяти.
Установка этого параметра велика может улучшить производительность базы данных, но слишком большое значение может привести к недостатку памяти для других процессов на сервере.
Рекомендуется устанавливать его на 25% - 40% доступной оперативной памяти на сервере.

work_mem = '32 MB'
Определяет объем памяти, выделенной для каждой операции сортировки или объединения (merge) в запросе. Это может быть, например, сортировка результатов запроса или выполнение операции объединения.
Увеличение этого параметра может ускорить выполнение запросов, которые используют сортировку или объединение, но слишком большое значение может привести к истощению памяти и увеличению времени выполнения запросов.
Рекомендуется настраивать его так, чтобы суммарный объем памяти, выделенный для всех параллельных операций, не превышал доступной оперативной памяти на сервере.

Кроме этих двух параметров, существует множество других, которые также могут повлиять на производительность базы данных PostgreSQL. Некоторые из них:

- effective_cache_size: Параметр, указывающий PostgreSQL оценку размера кэша файловой системы операционной системы. Это позволяет PostgreSQL лучше использовать кэш файловой системы для чтения данных с диска.
- maintenance_work_mem: Определяет объем памяти, выделенный для операций обслуживания базы данных, таких как создание индексов или анализ таблиц.
- checkpoint_completion_target: Определяет, какую часть цели чекпоинта должен завершить каждый чекпоинт, что влияет на частоту и продолжительность чекпоинтов.
- autovacuum_work_mem: Определяет объем памяти, выделенный для операций автоочистки (autovacuum), которые поддерживают целостность данных и оптимизируют производительность.
- max_connections: Определяет максимальное количество одновременных соединений с базой данных.
