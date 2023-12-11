### **Домашнее задание 1**
#### *Работа с уровнями изоляции транзакции в PostgreSQL*
-------------------------------------------------------
- virtual machine (Ubuntu 22.04.3 LTS) oracle vm virtaulbox
- postgresql 15
  
*session 1:*
*Подключаемся на сервер первой сессией*
```
# sudo -u postgres psql
postgres=#
```
*Выключаем auto commit*
```sql
\set AUTOCOMMIT off
```
*Создаем новую таблицу и заполняем данными с подтверждением транзакции*
```sql
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
````
*Подтверждаем транзакцию*
```sql
commit;
```
```sql
postgres=# create table persons(id serial, first_name text, second_name text);
CREATE TABLE
postgres=*# insert into persons(first_name, second_name) values('ivan', 'ivanov');
INSERT 0 1
postgres=*# insert into persons(first_name, second_name) values('petr', 'petrov');
INSERT 0 1
postgres=*# commit;
COMMIT
```
*Проверяем текущий уровень изолции*
```sql
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
*session 2:*
*Подключаемся на сервер второй сессией*
```
# sudo -u postgres psql
postgres=#
```
*Выключаем auto commit*
```sql
\set AUTOCOMMIT off
```
*Проверяем текущий уровень изолции*
```sql
postgres=# show transaction isolation level;
 transaction_isolation
-----------------------
 read committed
(1 row)
```
*Запускаем в обеих сессиях транзакции с дефолтным уровенм изоляции*
```sql
BEGIN TRANSACTION ISOLATION LEVEL READ COMMITTED;
```
*В первой сессии добавляем запись в таблицу*
```sql
insert into persons(first_name, second_name) values('sergey', 'sergeev');
postgres=*# insert into persons(first_name, second_name) values('sergey', 'sergeev');
INSERT 0 1
```
*Делаем выборку данных из таблцы во второй сессии*
```sql
select * from persons;
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```
**Данных не видно по причине того что транзакция еще не завершилась при уровне изоляции read committed в первой сессии**

*Завершаем транзакцию в первой сессии*
```sql
commit;
postgres=*# commit;
COMMIT
```
*Делаем выборку данных из таблцы во второй сессии*
```sql
select * from persons;
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
**Данные появились, так как транзакция была завершена в первой сессии**

*Начианем транзакции с уровнем изоляции в обоих сессиях repeatable*
```sql
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
postgres=# BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN
postgres=*#
```
*Добавляем новую запись в первой сессии*
```sql
insert into persons(first_name, second_name) values('sveta', 'svetova');
postgres=*# insert into persons(first_name, second_name) values('sveta', 'svetova');
INSERT 0 1
```
*Совершаем выборку данных во второй сесии*
```sql
select* from persons;
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
*Новой записи нет, завершаем первую транзакцию*
```sql
postgres=*# commit;
COMMIT
```
*Повторно делаем выборку данных во второй сессии*
```sql
postgres=*# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
(3 rows)
```
*Данных по-прежнему нет, завершаем транзакцию во второй сессии и повторяем выборку*
```sql
postgres=*# commit;
COMMIT
postgres=# select * from persons;
 id | first_name | second_name
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev
  4 | sveta      | svetova
(4 rows)
```
**Данные появились после завершения всех транзкций (в двух сессиях), в режиме изоляции REPEATABLE**
