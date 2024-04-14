### **Домашнее задание 9**
#### *Бэкапы*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

Цель:
Применить логический бэкап. Восстановиться из бэкапа.
-------------------------------------------------------
*Виртуальная машина и PostgreSQL развернуты аналогичным способом, уже пердустановлены*

*Создаем БД, схему и в ней таблицу*

```bash
root@otus-edu-pg-vm-1:~ # su postgres
postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=#
postgres=# create database otus_backup;
CREATE DATABASE
postgres=# \c otus_backup;
You are now connected to database "otus_backup" as user "postgres".
otus_backup=# create schema backup;
CREATE SCHEMA
otus_backup=# create table backup.backup_tbl (id int PRIMARY KEY, text text);
CREATE TABLE
otus_backup=#
```

*Заполним таблицы автосгенерированными 100 записями*
```bash
otus_backup=# insert into backup.backup_tbl values (generate_series(1, 100), md5(random()::text));
INSERT 0 100
otus_backup=# select * from backup.backup_tbl limit 33;
 id |               text
----+----------------------------------
  1 | 74b2e101ee3260f1088ef7ccdeb99501
  2 | d0eeda6993c69febdc23380e4dd94d12
  3 | 0ecc3a15433727d3c43a2d80cca5483f
  4 | 26060d8d84016049346e77cda4019273
  5 | fa6d7221546415d30199bc0470648e79
  6 | 67b5ac996ba55bbc1836938e5d5733a1
  7 | ebb9104767df8c67e18c33d0bec4f8e6
  8 | 6f704755815c9ec1ba2f29774493698d
  9 | da96c74682bf292a29f8247b8fea9b3b
 10 | df7af6de167266f39a85fe8ea1139e25
 11 | ae28342edbaca15e074e48e178eabb04
 12 | 5865cc0271d4a0b12233bf16a68d57e7
 13 | 2294aacfd35618a8cbb0523a82e7cb34
 14 | 37cb98ff22dbb2f6c7705a2f3db18a45
 15 | f215075eefa11c213ab6d6337a3cf118
 16 | 60ed56ed0951f3f63fb3ff38f0bc6fe8
 17 | bf5e03d859c94cc8105025ad4e76c00b
 18 | 06b400f5f326aa6c14c2732dfc588880
 19 | 8f46b7cca9f497120e5e93e6747ccab5
 20 | 99495a455a07ef7ced84a8cf5c79257e
 21 | 0feaa3081e575032893e8594a0269aee
 22 | 704d12e744f8a7cb21d0b845753eae73
 23 | 99b1b33a3651fe859f872a0585118b45
 24 | 004c8bc25df3560680daf0a5a5290b1d
 25 | 35982a1cc3b3aff095c53399b50a3c53
 26 | f79238abf0c558ccf8adcff075b87f67
 27 | b677201a16d8c2587cf712e0287375ff
 28 | ec19645e991f7cd7f9cf3d4fb85ede26
 29 | ad217ae30d34ea8d4186a7818a7949d9
 30 | 6d57287cea7a5ca27316dc24be3e420e
 31 | 362faf2fbc3a6c84e0ae756305df1a05
 32 | 296b8ab4f382caa1c187776f6a221ee7
 33 | 1bcddd7116a0d995127368dbf60d293e
(33 rows)
otus_backup=#
```

*Под линукс пользователем Postgres создадим каталог для бэкапов*
```bash
root@otus-edu-pg-vm-1:~ # mkdir backups
root@otus-edu-pg-vm-1:~ #
```

*Сделаем логический бэкап используя утилиту COPY*
```bash
root@otus-edu-pg-vm-1:~ # su postgres
postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c otus_backup
You are now connected to database "otus_backup" as user "postgres".
otus_backup=# copy backup.backup_tbl to '/root/backups/backup_tbl.sql' (DELIMITER ',');
COPY 100
otus_backup=#
```

*Восстановим в 2 таблицу данные из бэкапа*
```bash
root@otus-edu-pg-vm-1:~ # su postgres
postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.
postgres=# \c otus_backup
You are now connected to database "otus_backup" as user "postgres".
otus_backup=# create table backup.backup_tbl_2 (id integer primary key, text text);
CREATE TABLE
otus_backup=# copy backup.backup_tbl_2 from '/root/backups/backup_tbl.sql' (DELIMITER ',');
COPY 100
otus_backup=# select * from backup.backup_tbl_2 limit 15;
 id |               text
----+----------------------------------
  1 | 74b2e101ee3260f1088ef7ccdeb99501
  2 | d0eeda6993c69febdc23380e4dd94d12
  3 | 0ecc3a15433727d3c43a2d80cca5483f
  4 | 26060d8d84016049346e77cda4019273
  5 | fa6d7221546415d30199bc0470648e79
  6 | 67b5ac996ba55bbc1836938e5d5733a1
  7 | ebb9104767df8c67e18c33d0bec4f8e6
  8 | 6f704755815c9ec1ba2f29774493698d
  9 | da96c74682bf292a29f8247b8fea9b3b
 10 | df7af6de167266f39a85fe8ea1139e25
 11 | ae28342edbaca15e074e48e178eabb04
 12 | 5865cc0271d4a0b12233bf16a68d57e7
 13 | 2294aacfd35618a8cbb0523a82e7cb34
 14 | 37cb98ff22dbb2f6c7705a2f3db18a45
 15 | f215075eefa11c213ab6d6337a3cf118
(15 rows)

otus_backup=#
```

*Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц*
```bash
postgres@otus-edu-pg-vm-1:/root$ pg_dump --dbname=otus_backup --schema=backup --table=backup.backup_tbl* --format=c --compress=6 --file=pgdump_backup.sql
postgres@otus-edu-pg-vm-1:/root$
```

*Используя утилиту pg_restore восстановим в новую БД только вторую таблицу*
```bash
postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database otus_backup2;
CREATE DATABASE
postgres=# \c otus_backup2;
You are now connected to database "otus_backup2" as user "postgres".
otus_backup2=# create schema backup;
CREATE SCHEMA
otus_backup2=# \q
postgres@otus-edu-pg-vm-1:/root$ pg_restore --dbname=otus_backup2 --table=backup_tbl_2 pgdump_backup.sql

postgres@otus-edu-pg-vm-1:/root$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \c otus_backup2;
You are now connected to database "otus_backup2" as user "postgres".
otus_backup2=# \dt backup.*
            List of relations
 Schema |     Name     | Type  |  Owner
--------+--------------+-------+----------
 backup | backup_tbl_2 | table | postgres
(1 row)

otus_backup2=# select * from backup.backup_tbl_2 limit 5;
 id |               text
----+----------------------------------
  1 | 74b2e101ee3260f1088ef7ccdeb99501
  2 | d0eeda6993c69febdc23380e4dd94d12
  3 | 0ecc3a15433727d3c43a2d80cca5483f
  4 | 26060d8d84016049346e77cda4019273
  5 | fa6d7221546415d30199bc0470648e79
(5 rows)

otus_backup2=#
```

Все прошло успешно
