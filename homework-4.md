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
root@otus-pg-edu-4:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
*Проверяем кластер*
```bash
root@otus-pg-edu-4:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
root@otus-pg-edu-4:~#
```
