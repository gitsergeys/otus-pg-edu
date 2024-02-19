### **Домашнее задание 7**
#### *Механизм блокировок*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

*Проверяем и настраиваем сервер так, чтобы в журнал сообщений (логирование) сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд.*
```bash
postgres@otus-edu-pg-vm-1:~$ psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# show log_lock_waits;
 log_lock_waits
----------------
 off
(1 row)

postgres=# show deadlock_timeout;
 deadlock_timeout
------------------
 1s
(1 row)

postgres=# alter system set log_lock_waits = on;
ALTER SYSTEM
postgres=# alter system set deadlock_timeout = 200;
ALTER SYSTEM
postgres=# select pg_reload_conf();
 pg_reload_conf
----------------
 t
(1 row)

postgres=#
```
*Подготавливаем и воспроизводим ситуацию.*
```bash
postgres=# create table otus_locks (id bigserial primary key, amount integer);
CREATE TABLE
postgres=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | otus_locks | table | postgres
(1 row)

postgres=# insert into otus_locks (amount) values (111), (222), (333);
INSERT 0 3
postgres=# select * from otus_locks;
 id | amount
----+--------
  1 |    111
  2 |    222
  3 |    333
(3 rows)

-- В первом подключении
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 444 where id = 1;
UPDATE 1
postgres=*#

-- Во  втором подключении
postgres=# begin;
BEGIN
postgres=# update otus_locks set amount = 555 where id = 1;
-- блокировака
```
*Смотрим инфомарцию в логах и видим что произошла блокировка более 200 мс ожидания process 3337 still waiting for ShareLock on transaction 737 after 200.960 ms*
```bash
root@otus-edu-pg-vm-1:~ # tail -f -n 4 /var/log/postgresql/postgresql-15-main.log
2024-02-19 15:52:33.195 MSK [3337] postgres@postgres LOG:  process 3337 still waiting for ShareLock on transaction 737 after 200.960 ms
2024-02-19 15:52:33.195 MSK [3337] postgres@postgres DETAIL:  Process holding the lock: 3154. Wait queue: 3337.
2024-02-19 15:52:33.195 MSK [3337] postgres@postgres CONTEXT:  while updating tuple (0,1) in relation "otus_locks"
2024-02-19 15:52:33.195 MSK [3337] postgres@postgres STATEMENT:  update otus_locks set amount = 555 where id = 1;
```
*Делаем commits по очереди, начиная с первого подключения для решения вопроса*
```bash
-- первая
postgres=*# commit;
COMMIT
postgres=#
-- во второv update прошел сразу же и делаем аналогично commit
postgres=# update otus_locks set amount = 555 where id = 1;
UPDATE 1
postgres=# commit;
COMMIT
postgres=# select * from otus_locks;
 id | amount
----+--------
  2 |    222
  3 |    333
  1 |    555
(3 rows)
```
*Смоделируем ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах.*
```bash
-- первая
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 1 where id = 1;
UPDATE 1
postgres=*#
-- вторая
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 2 where id = 1;
...блокировка
-- третья
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 3 where id = 1;
...блокировка
```
*Изучаем возникшие блокировки в представлении pg_locks. Пришлите список блокировок и объясните, что значит каждая.*
*После анализа и изучения, для убодного вида составлен следующий итоговый запрос*
```bash
postgres=*# select locktype, relation::regclass, mode, pid, pg_blocking_pids(pid) from pg_locks where relation = 'otus_locks'::regclass and locktype = 'relation';
 locktype |  relation  |       mode       | pid  | pg_blocking_pids
----------+------------+------------------+------+------------------
 relation | otus_locks | RowExclusiveLock | 3655 | {3337}
 relation | otus_locks | RowExclusiveLock | 3337 | {3154}
 relation | otus_locks | RowExclusiveLock | 3154 | {}
(3 rows)

postgres=*#
```
*Наблюдаем что есть три блоккировки RowExclusiveLock (на update) а так же, что: первая сессия с pid 3154 заблокировала вторую сессию 3337, вторая сессия 3337 заблокировала третью сессию 3655*

*Производим поочередные commits с первой по третью сессии для решения вопроса и проверяем что в итоге записалось в таблице*
```bash
postgres=*# commit;
COMMIT
postgres=# select * from otus_locks;
 id | amount
----+--------
  2 |    222
  3 |    333
  1 |      3
(3 rows)

postgres=#
```
*Воспроизводим взаимоблокировку трех транзакций, соответственно аналогично в трех сессиях*
```bash
-- первая
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 001 where id = 1;
UPDATE 1
postgres=*#

-- вторая
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 002 where id = 2;
UPDATE 1
postgres=*#

-- третья
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 003 where id = 3;
UPDATE 1
postgres=*#
```
*Не завершая все 3 транзакции пытаемся обновить данные по этим же полям*
```bash
-- первая
postgres=*# update otus_locks set amount = 301 where id = 3;
...блокировка

-- вторая
postgres=*# update otus_locks set amount = 201 where id = 1;
..блокировка

-- третья
postgres=*# update otus_locks set amount = 101 where id = 1;
...блокировка

-- первая сессия получили ошибку, делаем rollback
ERROR:  deadlock detected
DETAIL:  Process 3154 waits for ShareLock on transaction 751; blocked by process 3655.
Process 3655 waits for ExclusiveLock on tuple (0,15) of relation 24577 of database 5; blocked by process 3337.
Process 3337 waits for ShareLock on transaction 749; blocked by process 3154.
HINT:  See server log for query details.
CONTEXT:  while updating tuple (0,3) in relation "otus_locks"
postgres=!# rollback;
ROLLBACK

-- вторая сессия обновилась
postgres=*# update otus_locks set amount = 201 where id = 1;
UPDATE 1
postgres=*#

-- третья сессия висит, делаем commit во второй сессии
postgres=*# commit;
COMMIT
postgres=#

-- третья сессия прошла без блокировки, делаем commit
postgres=*# update otus_locks set amount = 101 where id = 1;
UPDATE 1
postgres=*#

-- проверяем данные в таблице
postgres=# select * from otus_locks;
 id | amount
----+--------
  2 |      2
  3 |      3
  1 |    101
(3 rows)
```
*В итоге обновилась 1 строка в последней сессии, так как я случайно прописал в sql запросе одинаковые id = 1, по идее должно было обновиться две строки т.е. во второй сессии и третьей, кроме первой которая в deadlocke отбилась :)*

*Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?*

Да можно, вся информация есть в логе:
```bash
2024-02-19 17:06:24.896 MSK [3154] postgres@postgres DETAIL:  Process holding the lock: 3655. Wait queue: .
2024-02-19 17:06:24.896 MSK [3154] postgres@postgres CONTEXT:  while updating tuple (0,3) in relation "otus_locks"
2024-02-19 17:06:24.896 MSK [3154] postgres@postgres STATEMENT:  update otus_locks set amount = 301 where id = 3;
2024-02-19 17:06:24.897 MSK [3154] postgres@postgres ERROR:  deadlock detected
2024-02-19 17:06:24.897 MSK [3154] postgres@postgres DETAIL:  Process 3154 waits for ShareLock on transaction 751; blocked by process 3655.
        Process 3655 waits for ExclusiveLock on tuple (0,15) of relation 24577 of database 5; blocked by process 3337.
        Process 3337 waits for ShareLock on transaction 749; blocked by process 3154.
        Process 3154: update otus_locks set amount = 301 where id = 3;
        Process 3655: update otus_locks set amount = 101 where id = 1;
        Process 3337: update otus_locks set amount = 201 where id = 1;
```
*Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?*

Думаю да, попробем проверить на практике:

```bash
-- первая сессия
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 007;
UPDATE 3
postgres=*#

-- вторая сессия
postgres=# begin;
BEGIN
postgres=*# update otus_locks set amount = 777;
...блокировка

-- делаем commit первой сессии
postgres=*# commit;
COMMIT

-- вторая прошла :) делаем commit
UPDATE 3
postgres=*# commit;
COMMIT
postgres=# select * from otus_locks;
 id | amount
----+--------
  2 |    777
  3 |    777
  1 |    777
(3 rows)

postgres=#
```
Да, блокируют
