### **Домашнее задание 6**
#### *Работа с журналами*
-------------------------------------------------------
- virtual machine 2CPU, 4GB RAM, 15GB SATA HDD (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15
- postgresql 14

*Подключаемся на VM и устанавливаем PostgreSQL 15, обновляемся*
```bash
script@otus-pg-edu-vm:~$ sudo -i
root@otus-pg-edu-vm:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
