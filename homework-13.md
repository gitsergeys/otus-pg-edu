### **Домашнее задание 13**
#### *Секционирование таблицы*

> ## Секционировать большую таблицу из демо базы flights
* Загружаем большую БД с полетами за год [demo-big](https://edu.postgrespro.ru/demo-big.zip)
    <pre><details><summary>Вывод терминала</summary>
    ubuntu@pg-srv:~$ cd /tmp/
    ubuntu@pg-srv:/tmp$ 
    ubuntu@pg-srv:/tmp$ wget https://edu.postgrespro.ru/demo-big.zip
    --2023-06-28 20:42:55--  https://edu.postgrespro.ru/demo-big.zip
    Resolving edu.postgrespro.ru (edu.postgrespro.ru)... 213.171.56.196
    Connecting to edu.postgrespro.ru (edu.postgrespro.ru)|213.171.56.196|:443... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 243203214 (232M) [application/zip]
    Saving to: ‘demo-big.zip’
    
    demo-big.zip                                               100%[========================================================================================================================================>] 231.94M  22.3MB/s    in 11s     
    
    2023-06-28 20:43:05 (22.1 MB/s) - ‘demo-big.zip’ saved [243203214/243203214]
    
    ubuntu@pg-srv:/tmp$ 
    ubuntu@pg-srv:/tmp$ chmod 777 /tmp/demo-big.zip 
    ubuntu@pg-srv:/tmp$ 
    ubuntu@pg-srv:/tmp$ unzip demo-big.zip 
    Archive:  demo-big.zip
      inflating: demo-big-20170815.sql   
    ubuntu@pg-srv:/tmp$ 
    ubuntu@pg-srv:/tmp$ ls -ltr
    total 1146768
    -rw-rw-r-- 1 ubuntu ubuntu 931068524 Jan  5  2018 demo-big-20170815.sql
    -rwxrwxrwx 1 ubuntu ubuntu 243203214 Jan 11  2018 demo-big.zip
    drwx------ 3 root   root        4096 Jun 28 20:16 systemd-private-52a2987604b24b48975b7e85755c6c51-systemd-timesyncd.service-32aSZi
    drwx------ 3 root   root        4096 Jun 28 20:16 systemd-private-52a2987604b24b48975b7e85755c6c51-systemd-resolved.service-Tqv8dj
    drwx------ 3 root   root        4096 Jun 28 20:16 systemd-private-52a2987604b24b48975b7e85755c6c51-systemd-logind.service-KJrn8e
    drwxr-xr-x 2 root   root        4096 Jun 28 20:20 ubuntu-advantage
    ubuntu@pg-srv:/tmp$ 
    ubuntu@pg-srv:/tmp$ sudo -u postgres psql -f demo-big-20170815.sql 
    SET
    SET
    SET
    SET
    SET
    SET
    SET
    SET
    psql:demo-big-20170815.sql:17: ERROR:  database "demo" does not exist
    CREATE DATABASE
    You are now connected to database "demo" as user "postgres".
    SET
    SET
    SET
    SET
    SET
    SET
    SET
    SET
    CREATE SCHEMA
    COMMENT
    CREATE EXTENSION
    COMMENT
    SET
    CREATE FUNCTION
    CREATE FUNCTION
    COMMENT
    SET
    SET
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE VIEW
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE VIEW
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE SEQUENCE
    ALTER SEQUENCE
    CREATE VIEW
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE VIEW
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    CREATE TABLE
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    COMMENT
    ALTER TABLE
    COPY 9
    COPY 104
    COPY 7925812
    COPY 2111110
    COPY 214867
    COPY 1339
    COPY 8391852
    COPY 2949857
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER TABLE
    ALTER DATABASE
    ALTER DATABASE
    ubuntu@pg-srv:/tmp$ 
    </details></pre>
  * Смотрим, какой диапазон данных(mix, max) есть в таблице `flights`
  ```pgsql
  demo=# select min(scheduled_departure),max(scheduled_departure) from flights;
            min           |          max           
  ------------------------+------------------------
   2016-08-14 23:45:00+00 | 2017-09-14 17:55:00+00
  (1 row)
  ```

  * Создаем партицированную таблицу `flights_new`, с ключом по диапазону `range` по полю `scheduled_departure`+партиции
  ```pgsql
  demo=# create table flights_new (like flights) partition by range (scheduled_departure);
  CREATE TABLE
  demo=# 
  demo=# create table flights_new_2016_08 partition of flights_new for values from ('2016-08-01') to ('2016-09-01');
  CREATE TABLE
  demo=# create table flights_new_2016_09 partition of flights_new for values from ('2016-09-01') to ('2016-10-01');
  CREATE TABLE
  demo=# create table flights_new_2016_10 partition of flights_new for values from ('2016-10-01') to ('2016-11-01');
  CREATE TABLE
  demo=# create table flights_new_2016_11 partition of flights_new for values from ('2016-11-01') to ('2016-12-01');
  CREATE TABLE
  demo=# create table flights_new_2016_12 partition of flights_new for values from ('2016-12-01') to ('2017-01-01');
  CREATE TABLE
  demo=# create table flights_new_2017_01 partition of flights_new for values from ('2017-01-01') to ('2017-02-01');
  CREATE TABLE
  demo=# create table flights_new_2017_02 partition of flights_new for values from ('2017-02-01') to ('2017-03-01');
  CREATE TABLE
  demo=# create table flights_new_2017_03 partition of flights_new for values from ('2017-03-01') to ('2017-04-01');
  CREATE TABLE
  demo=# create table flights_new_2017_04 partition of flights_new for values from ('2017-04-01') to ('2017-05-01');
  CREATE TABLE
  demo=# create table flights_new_2017_05 partition of flights_new for values from ('2017-05-01') to ('2017-06-01');
  CREATE TABLE
  demo=# create table flights_new_2017_06 partition of flights_new for values from ('2017-06-01') to ('2017-07-01');
  CREATE TABLE
  demo=# create table flights_new_2017_07 partition of flights_new for values from ('2017-07-01') to ('2017-08-01');
  CREATE TABLE
  demo=# create table flights_new_2017_08 partition of flights_new for values from ('2017-08-01') to ('2017-09-01');
  CREATE TABLE
  demo=# create table flights_new_2017_09 partition of flights_new for values from ('2017-09-01') to ('2017-10-01');
  CREATE TABLE
  demo=# 
  demo=# 
  demo=# \d+ flights_new
                                                 Partitioned table "bookings.flights_new"
         Column        |           Type           | Collation | Nullable | Default | Storage  | Compression | Stats target | Description 
  ---------------------+--------------------------+-----------+----------+---------+----------+-------------+--------------+-------------
   flight_id           | integer                  |           | not null |         | plain    |             |              | 
   flight_no           | character(6)             |           | not null |         | extended |             |              | 
   scheduled_departure | timestamp with time zone |           | not null |         | plain    |             |              | 
   scheduled_arrival   | timestamp with time zone |           | not null |         | plain    |             |              | 
   departure_airport   | character(3)             |           | not null |         | extended |             |              | 
   arrival_airport     | character(3)             |           | not null |         | extended |             |              | 
   status              | character varying(20)    |           | not null |         | extended |             |              | 
   aircraft_code       | character(3)             |           | not null |         | extended |             |              | 
   actual_departure    | timestamp with time zone |           |          |         | plain    |             |              | 
   actual_arrival      | timestamp with time zone |           |          |         | plain    |             |              | 
  Partition key: RANGE (scheduled_departure)
  Partitions: flights_new_2016_08 FOR VALUES FROM ('2016-08-01 00:00:00+00') TO ('2016-09-01 00:00:00+00'),
              flights_new_2016_09 FOR VALUES FROM ('2016-09-01 00:00:00+00') TO ('2016-10-01 00:00:00+00'),
              flights_new_2016_10 FOR VALUES FROM ('2016-10-01 00:00:00+00') TO ('2016-11-01 00:00:00+00'),
              flights_new_2016_11 FOR VALUES FROM ('2016-11-01 00:00:00+00') TO ('2016-12-01 00:00:00+00'),
              flights_new_2016_12 FOR VALUES FROM ('2016-12-01 00:00:00+00') TO ('2017-01-01 00:00:00+00'),
              flights_new_2017_01 FOR VALUES FROM ('2017-01-01 00:00:00+00') TO ('2017-02-01 00:00:00+00'),
              flights_new_2017_02 FOR VALUES FROM ('2017-02-01 00:00:00+00') TO ('2017-03-01 00:00:00+00'),
              flights_new_2017_03 FOR VALUES FROM ('2017-03-01 00:00:00+00') TO ('2017-04-01 00:00:00+00'),
              flights_new_2017_04 FOR VALUES FROM ('2017-04-01 00:00:00+00') TO ('2017-05-01 00:00:00+00'),
              flights_new_2017_05 FOR VALUES FROM ('2017-05-01 00:00:00+00') TO ('2017-06-01 00:00:00+00'),
              flights_new_2017_06 FOR VALUES FROM ('2017-06-01 00:00:00+00') TO ('2017-07-01 00:00:00+00'),
              flights_new_2017_07 FOR VALUES FROM ('2017-07-01 00:00:00+00') TO ('2017-08-01 00:00:00+00'),
              flights_new_2017_08 FOR VALUES FROM ('2017-08-01 00:00:00+00') TO ('2017-09-01 00:00:00+00'),
              flights_new_2017_09 FOR VALUES FROM ('2017-09-01 00:00:00+00') TO ('2017-10-01 00:00:00+00')
  
  demo=# 
  ```
  * Заливаем данные из таблицы `flights` в таблицу `flights_new` и смотрим размер
  ```pgsql
  demo=# insert into flights_new (select * from flights);
  INSERT 0 214867
  demo=# 
  demo=# 
  demo=# select 'select pg_size_pretty(pg_table_size('''||tablename||'''));' from pg_tables where tablename like 'flights_new%';
                             ?column?                           
  --------------------------------------------------------------
   select pg_size_pretty(pg_table_size('flights_new'));
   select pg_size_pretty(pg_table_size('flights_new_2016_08'));
   select pg_size_pretty(pg_table_size('flights_new_2016_09'));
   select pg_size_pretty(pg_table_size('flights_new_2016_10'));
   select pg_size_pretty(pg_table_size('flights_new_2016_11'));
   select pg_size_pretty(pg_table_size('flights_new_2016_12'));
   select pg_size_pretty(pg_table_size('flights_new_2017_01'));
   select pg_size_pretty(pg_table_size('flights_new_2017_02'));
   select pg_size_pretty(pg_table_size('flights_new_2017_03'));
   select pg_size_pretty(pg_table_size('flights_new_2017_04'));
   select pg_size_pretty(pg_table_size('flights_new_2017_05'));
   select pg_size_pretty(pg_table_size('flights_new_2017_06'));
   select pg_size_pretty(pg_table_size('flights_new_2017_07'));
   select pg_size_pretty(pg_table_size('flights_new_2017_08'));
   select pg_size_pretty(pg_table_size('flights_new_2017_09'));
  (15 rows)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new'));
   pg_size_pretty 
  ----------------
   0 bytes
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2016_08'));
   pg_size_pretty 
  ----------------
   944 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2016_09'));
   pg_size_pretty 
  ----------------
   1640 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2016_10'));
   pg_size_pretty 
  ----------------
   1704 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2016_11'));
   pg_size_pretty 
  ----------------
   1648 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2016_12'));
   pg_size_pretty 
  ----------------
   1696 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_01'));
   pg_size_pretty 
  ----------------
   1696 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_02'));
   pg_size_pretty 
  ----------------
   1536 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_03'));
   pg_size_pretty 
  ----------------
   1696 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_04'));
   pg_size_pretty 
  ----------------
   1648 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_05'));
   pg_size_pretty 
  ----------------
   1696 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_06'));
   pg_size_pretty 
  ----------------
   1640 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_07'));
   pg_size_pretty 
  ----------------
   1704 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_08'));
   pg_size_pretty 
  ----------------
   1624 kB
  (1 row)
  
  demo=#  select pg_size_pretty(pg_table_size('flights_new_2017_09'));
   pg_size_pretty 
  ----------------
   728 kB
  (1 row)
  
  demo=# 
  ```

  * Видим, что основная таблица не соддержит данных, а партиции-содержат
  * Чтобы работал режим выборки данных из партиций, надо чтобы параметры [enable_partition_pruning](https://postgrespro.ru/docs/postgresql/15/runtime-config-query#GUC-ENABLE-PARTITION-PRUNING) был равен `on`
  * В реальной жизни, я бы создал партицированную таблицу и порциями заливал туда данные из непартиционированной.
  * Не хватает split и merge partition в стандартной реализации postgre
