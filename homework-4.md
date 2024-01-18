### **Домашнее задание 4**
#### *Логический уровень: Работа с базами данных, пользователями и правами*
-------------------------------------------------------
- virtual machine (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

*Подключаемся на VM*
```bash
script@otus-pg-edu-4:~# sudo -i
root@otus-pg-edu-4:~#
```
*Устанавливаем PostgreSQL 15 и обновляемся*
```bash
root@otus-pg-edu-4:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15 -> #1
```
*Решение*

```bash
root@otus-pg-edu-4:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
root@otus-pg-edu-4:~# 
root@otus-pg-edu-4:~# su postgres -> #2
postgres@otus-pg-edu-4:/root$ cd ~
postgres@otus-pg-edu-4:~$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create database testdb; -> #3
CREATE DATABASE
postgres=# \c testdb -> #4
You are now connected to database "testdb" as user "postgres".
testdb=# create schema testnm; -> #5
CREATE SCHEMA
testdb=# create table t1 (c1 int); -> #6
CREATE TABLE
testdb=# insert into t1 values (1); -> #7
INSERT 0 1
testdb=# select * from t1;
 c1
----
  1
(1 row)
testdb=# create role readonly; -> #8
CREATE ROLE
testdb=# grant connect on database testdb to readonly; -> #9
GRANT
testdb=# grant usage on schema testnm to readonly; -> #10
GRANT
testdb=# grant select on all tables in schema testnm to readonly; -> #11
GRANT
testdb=# create user testread with password 'test123'; -> #12
CREATE ROLE
testdb=# grant readonly to testread; -> #13
GRANT ROLE
postgres=# \c testdb testread -> #14
You are now connected to database "testdb" as user "testread".
testdb=> select * from t1; -> #15
ERROR:  permission denied for table t1
testdb=> \dt -> #19
        List of relations
 Schema | Name | Type  |  Owner
--------+------+-------+----------
 public | t1   | table | postgres
(1 row)
testdb=> \c testdb postgres -> #22
You are now connected to database "testdb" as user "postgres".
testdb=# drop table t1; -> #23
DROP TABLE
testdb=# create table testnm.t1(c1 int); -> #24
CREATE TABLE
testdb=# insert into testnm.t1 values(1); -> #25
INSERT 0 1
testdb=# \c testdb testread -> #26
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1; -> #27
ERROR:  permission denied for table t1 -> #28
testdb=> \c testdb postgres; -> #29
You are now connected to database "testdb" as user "postgres".
testdb=# alter default privileges in schema testnm grant select on tables to readonly; -> #30
ALTER DEFAULT PRIVILEGES
testdb=# \c testdb testread 
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1;
ERROR:  permission denied for table t1
testdb=>  \c testdb postgres -> #30
testdb=# drop table testnm.t1; -> #30
DROP TABLE
testdb=# create table testnm.t1(c1 int); -> #30
CREATE TABLE
testdb=# insert into testnm.t1 values(1); -> #30
INSERT 0 1
testdb=#  \c testdb testread
You are now connected to database "testdb" as user "testread".
testdb=> select * from testnm.t1; -> #31
 c1
----
  1
(1 row)
testdb=> select current_user;
 current_user
--------------
 testread
(1 row)
--> #37 --> #42
testdb=> \c testdb postgres
You are now connected to database "testdb" as user "postgres".
testdb=# GRANT ALL ON schema testnm TO testread;
GRANT
testdb=# \c testdb testread
You are now connected to database "testdb" as user "testread".
testdb=> create table testnm.t2(c1 integer);
CREATE TABLE
testdb=> insert into testnm.t2 values(2);
INSERT 0 1
testdb=> select * from testnm.t2;
 c1
----
  2
(1 row)

testdb=> create table testnm.t3(c1 integer);
CREATE TABLE
testdb=> insert into testnm.t3 values(3);
INSERT 0 1
testdb=> select * from testnm.t3;
 c1
----
  3
(1 row)
<-- #37 <-- #42
testdb=>\q
postgres@otus-pg-edu-4:~$
```
