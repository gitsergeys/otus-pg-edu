### **Домашнее задание 10**
#### *Репликация*
-------------------------------------------------------
- 3x virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

Цель:
реализовать свой миникластер на 3 ВМ
-------------------------------------------------------
*Виртуальные машины и PostgreSQL развернуты аналогичным способом, уже пердустановлены*

### *Настройка логической репликации между 3 серверами*

> Создаем таблицы на чтения и запись на 3 VM (otus-edu-pg-vm-1, otus-edu-pg-vm-2, otus-edu-pg-vm-3).

```sql
-- Выполним команды на 3 серверах.
postgres=# create table test (info text PRIMARY KEY);
CREATE TABLE
postgres=# CREATE TABLE test2 (info text PRIMARY KEY);
CREATE TABLE

-- Заполним таблицу test на VM otus-edu-pg-vm-1 данными
postgres=# insert into test values ('otusedu1'), ('otusedu2'), ('otusedu3');
INSERT 0 3

-- Заполним таблицу test2 на VM otus-edu-pg-vm-2 данными
postgres=# insert into test2 values ('otusedu4'), ('otusedu5'), ('otusedu6');
INSERT 0 3
```

> Создаем публикации для таблиц test и test2

Настраиваем wal_level:

```sql
-- Для начала настроим wal_level на уровень logical на 3 VM
postgres=# alter system set wal_level = logical;
ALTER SYSTEM

-- Перезапустим сервис PostgreSQL на каждом из серверов
root@otus-edu-pg-vm-1:~ # systemctl restart postgresql@15-main
root@otus-edu-pg-vm-2:~ # systemctl restart postgresql@15-main
root@otus-edu-pg-vm-3:~ # systemctl restart postgresql@15-main

-- Проверим значение параметра wal_level
postgres=# show wal_level;
 wal_level
-----------
 logical
(1 row)
```

Создаем публикации:

```sql
-- otus-edu-pg-vm-1
postgres=# create publication test_pub_otus1 for table test;
CREATE PUBLICATION

-- otus-edu-pg-vm-2
postgres=# create publication test2_pub_otus2 for table test2;
CREATE PUBLICATION
```

> Создаем подписки
>
> 1. otus-edu-pg-vm-1 -> otus-edu-pg-vm-2;
> 2. otus-edu-pg-vm-2 -> otus-edu-pg-vm-1 (test);
> 3. otus-edu-pg-vm-1 <- otus-edu-pg-vm-3 -> otus-edu-pg-vm-2

**Подготовка**.

1. Меняем `listen_addresses` на `*`, чтобы прослушивать все адреса

```sql
-- Меняем значение параметра
postgres=# alter system set listen_addresses = '*';
ALTER SYSTEM

-- Перезапускаем сервер
root@otus-edu-pg-vm-1:~ # systemctl restart postgresql@15-main

-- Проверяем 5432
root@otus-edu-pg-vm-1:~ # ss -tlnup | grep 5432
tcp   LISTEN 0      244              0.0.0.0:5432       0.0.0.0:*    users:(("postgres",pid=572,fd=5))
tcp   LISTEN 0      244                 [::]:5432          [::]:*    users:(("postgres",pid=572,fd=6))
```

2. Устанавливаем пароль пользователя `postgres` на 3 VM:

```sql
postgres=# \password
Enter new password for user "postgres":
Enter it again:
```

3. Добавляем в `pg_hba.conf` запись на разрешение подключение:

VM `otus-edu-pg-vm-1`:

```ini
# Редактируем файл
root@otus-edu-pg-vm-1:~ # vim /etc/postgresql/15/main/pg_hba.conf

# Добавляем строчки
# otus-edu-pg-vm-2
host    postgres        postgres        192.168.1.32/32      trust
# otus-edu-pg-vm-3
host    postgres        postgres        192.168.1.33/32      trust

# Далее перечитаем конфигурацию
postgres=# select pg_reload_conf();
pg_reload_conf
----------------
 t
(1 row)
```

VM `otus-edu-pg-vm-2`:

```ini
# Редактируем файл
root@otus-edu-pg-vm-2:~ # vim /etc/postgresql/15/main/pg_hba.conf

# Добавляем строчки
# otus-edu-pg-vm-1
host    postgres        postgres        192.168.1.31/32      trust
# otus-edu-pg-vm-3
host    postgres        postgres        192.168.1.33/32      trust

# Далее перечитаем конфигурацию
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

VM `otus-edu-pg-vm-3`:

```ini
# Редактируем файл
root@otus-edu-pg-vm-3:~ # vim /etc/postgresql/15/main/pg_hba.conf

# Добавляем строчки
# otus-edu-pg-vm-1
host    postgres        postgres        192.168.1.31/32      trust
# otus-edu-pg-vm-2
host    postgres        postgres        192.168.1.32/32      trust

# Далее перечитаем конфигурацию
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)
```

*Создаем подписки*

1. *otus-edu-pg-vm-1 -> otus-edu-pg-vm-2*:

```sql
-- Проверим что таблица test2 пустая на VM otus-edu-pg-vm-1
postgres=# select * from test2;
 info
------
(0 rows)

-- Создаем подписку
postgres=# create subscription test2_sub_otus2 connection 'host=192.168.1.32 user=postgres dbname=postgres' publication test2_pub_otus2 with (copy_data = true);
CREATE SUBSCRIPTION

-- Проверияем данные
postgres=# select * from test2;
   info
----------
 otusedu4
 otusedu5
 otusedu6
(3 rows)

-- Смотрим статус репликации
postgres=# select * from pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16413
subname               | test2_sub_otus2
pid                   | 8062
relid                 |
received_lsn          | 0/156D690
last_msg_send_time    | 2024-04-23 11:28:45.59271+00
last_msg_receipt_time | 2024-04-23 11:28:45.593082+00
latest_end_lsn        | 0/156D690
latest_end_time       | 2024-04-23 11:28:45.59271+00
```

2. *otus-edu-pg-vm-2 -> otus-edu-pg-vm-1*

```sql
-- Проверим что таблица test пустая на VM otus-edu-pg-vm-2
postgres=# select * from test;
 info
------
(0 rows)

-- Создаем подписку
postgres=# create subscription test_sub_otus1 connection 'host=192.168.1.31 user=postgres dbname=postgres' publication test_pub_otus1 with (copy_data = true);
CREATE SUBSCRIPTION

-- Проверяем данные
postgres=# select * from test;
  info
---------
 otusedu1
 otusedu2
 otusedu3
(3 rows)

-- Смотрим на статус репликации
postgres=# select * from pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16404
subname               | test_sub_otus1
pid                   | 9590
relid                 |
received_lsn          | 0/157F3B0
last_msg_send_time    | 2024-04-23 11:35:15.209907+00
last_msg_receipt_time | 2024-04-23 11:35:15.209981+00
latest_end_lsn        | 0/157F3B0
latest_end_time       | 2024-04-23 11:35:15.209907+00
```

3. *otus-edu-pg-vm-1 <- otus-edu-pg-vm-3 -> otus-edu-pg-vm-2*

```sql
-- Проверяем что обе таблицы пустые на otus-edu-pg-vm-3
postgres=# select * from test;
 info
------
(0 rows)

postgres=# select * from test2;
 info
------
(0 rows)

-- Создаем подписки на таблицы test otus-edu-pg-vm-1 и test2 otus-edu-pg-vm-2
postgres=# create subscription test_sub_otus11 connection 'host=192.168.1.31 user=postgres dbname=postgres' publication test_pub_otus1 with (copy_data = true);
CREATE SUBSCRIPTION

postgres=# create subscription test2_sub_otus21 connection 'host=192.168.1.32 user=postgres dbname=postgres' publication test2_pub_otus2 with (copy_data = true);
CREATE SUBSCRIPTION

-- Проверяем данные в таблицах test и test2
postgres=# select * from test;
  info
---------
 otusedu1
 otusedu2
 otusedu3
(3 rows)

postgres=# select * from test2;
   info
----------
 otusedu4
 otusedu5
 otusedu6
(3 rows)

-- Проверяем статус репликации
postgres=# select * from pg_stat_subscription\gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16403
subname               | test_sub_otus11
pid                   | 10071
relid                 |
received_lsn          | 0/157F420
last_msg_send_time    | 2024-04-23 11:47:18.311463+00
last_msg_receipt_time | 2024-04-23 11:47:18.311419+00
latest_end_lsn        | 0/157F420
latest_end_time       | 2024-04-23 11:47:18.311463+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16404
subname               | test2_sub_otus21
pid                   | 10136
relid                 |
received_lsn          | 0/156E8D0
last_msg_send_time    | 2024-04-23 11:47:12.06938+00
last_msg_receipt_time | 2024-04-23 11:37:12.069466+00
latest_end_lsn        | 0/156E8D0
latest_end_time       | 2024-02-06 11:47:12.06938+00
```
