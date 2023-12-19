### **Домашнее задание 2**
#### *Установка и настройка PostgteSQL в контейнере Docker*
-------------------------------------------------------
- virtual machine (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15
- docker

*Устанавливаем docker (+curl)*
```bash
root@otus-pg-edu-2:~# apt install curl -y
root@otus-pg-edu-2:~# curl -fsSL https://get.docker.com -o get-docker.sh && sudo sh get-docker.sh && rm get-docker.sh && sudo usermod -aG docker $USER && newgrp docker
```
*Создаем docker-сеть*
```bash
root@otus-pg-edu-2:~# docker network create pg-net
e6211baa037e9b6f44f65df5a394f3254f3c602b0f853c66df5f0486ea5efbaf
root@otus-pg-edu-2:~#
```
*Создаем каталог postgres*
```bash
root@otus-pg-edu-2:~# mkdir /var/lib/postgres
root@otus-pg-edu-2:~# ls -l /var/lib/
...
drwxr-xr-x  2 root          docker        4096 дек 19 10:00 postgres
...
```
*Подключаем контейнер с PostgreSQL 15 смонтировав в /var/lib/postgresql*
```bash
root@otus-pg-edu-2:~# docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
Unable to find image 'postgres:15' locally
15: Pulling from library/postgres
1f7ce2fa46ab: Pull complete
9576f7629775: Pull complete
30655987a437: Pull complete
dea99268fe33: Pull complete
3bbfac6ba51f: Pull complete
e797e6a53a63: Pull complete
19e54d4f4a6c: Pull complete
ae10e0b63201: Pull complete
3eb0cb30d619: Pull complete
313add79a9a1: Pull complete
43025ad23358: Pull complete
475eb3b9678e: Pull complete
01e3503978ac: Pull complete
1a7db5aa4b09: Pull complete
Digest: sha256:bec340fb35711dd4a989146591b6adfaac53e6b0c02524ff956c43b054d117dd
Status: Downloaded newer image for postgres:15
d20455f1893b5bc4f1e57a9424450bf0d3cb4a0453d9faf66ed6323517c5e7b5
root@otus-pg-edu-2:~#
```
*Запускаем отдельный контейнер с клиентом в общей сети с БД, подключаемся и создаем таблицу с парой строк*
```bash
root@otus-pg-edu-2:~# docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# CREATE TABLE otus_table (id SERIAL PRIMARY KEY, first VARCHAR(10), second VARCHAR(10));
CREATE TABLE
postgres=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | otus_table | table | postgres
(1 row)

postgres=# INSERT INTO otus_table (first, second) VALUES ('record1', 'record2');
INSERT 0 1
postgres=# INSERT INTO otus_table (first, second) VALUES ('record3', 'record4');
INSERT 0 1
postgres=# SELECT * FROM otus_table;
 id |  first  | second
----+---------+---------
  1 | record1 | record2
  2 | record3 | record4
(2 rows)

postgres=#
```
*Подключаемся к контейнеру с сервером с ноутбука извне места установки докера*
- предварительно раскомментировав порт 5432 (*postgresql.conf*) на сервере и добавив в разрешенные IP адрес ноутбука, с которого будем подключаться (*pg_hba.conf*)
- так же потребуется предустановленный psql shell на ноутбуке
```bash
Server [localhost]: 192.168.1.145
Database [postgres]:
Port [5432]:
Username [postgres]:
Password for user postgres:
psql (15.5)
WARNING: Console code page (866) differs from Windows code page (1251)
         8-bit characters might not work correctly. See psql reference
         page "Notes for Windows users" for details.
Type "help" for help.

postgres=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | otus_table | table | postgres
(1 row)

postgres=#
```
*Удаляем контейнер с сервером*
```bash
root@otus-pg-edu-2:~# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
d20455f1893b   postgres:15   "docker-entrypoint.s…"   45 minutes ago   Up 45 minutes   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
root@otus-pg-edu-2:~# docker stop d20455f1893b
d20455f1893b
root@otus-pg-edu-2:~# docker rm d20455f1893b
d20455f1893b
root@otus-pg-edu-2:~# docker ps -a
CONTAINER ID   IMAGE     COMMAND   CREATED   STATUS    PORTS     NAMES
root@otus-pg-edu-2:~#
```
*Создаем контейнер заново*
```bash
root@otus-pg-edu-2:~# docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
9c1d48d0cd87c1b945bc3a383ed20b4ae9f5e9da8aeb3a051a1f131f1fea8245
root@otus-pg-edu-2:~# docker ps -a
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                                       NAMES
9c1d48d0cd87   postgres:15   "docker-entrypoint.s…"   16 seconds ago   Up 15 seconds   0.0.0.0:5432->5432/tcp, :::5432->5432/tcp   pg-server
root@otus-pg-edu-2:~#
```
*Подключаемся снова из контейнера с клиентом к контейнеру с сервером и проверяем что данные сохранились*
```bash
root@otus-pg-edu-2:~# docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
Password for user postgres:
psql (15.5 (Debian 15.5-1.pgdg120+1))
Type "help" for help.

postgres=# \dt
           List of relations
 Schema |    Name    | Type  |  Owner
--------+------------+-------+----------
 public | otus_table | table | postgres
(1 row)

postgres=#
```

**Все данные в БД на месте, после удаления и создания контейнеров**
