### **Домашнее задание 3**
#### *Физический уровень: Установка и настройка PostgreSQL*
-------------------------------------------------------
- virtual machine (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- postgresql 15

*Подключаемся на VM*
```bash
script@otus-pg-edu-3:~$ sudo -i
root@otus-pg-edu-3:~#
```
*Устанавливаем PostgreSQL 15 и обновляемся*
```bash
root@otus-pg-edu-3:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
*Проверяем кластер*
```bash
root@otus-pg-edu-3:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
root@otus-pg-edu-3:~#
```
*Подключаемся в psql пользователем postgres и создаем таблицу с данными*
```bash
root@otus-pg-edu-3:~# sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# create table table_1(test_info text);
CREATE TABLE
postgres=# insert into table_1 values('otus_education');
INSERT 0 1
postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | table_1 | table | postgres
(1 row)

postgres=# select * from table_1;
   test_info
----------------
 otus_education
(1 row)

postgres=# \q
root@otus-pg-edu-3:~#
```
*Останавливаем кластер*
```bash
root@otus-pg-edu-3:~# pg_ctlcluster 15 main stop
root@otus-pg-edu-3:~# pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 down   postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
root@otus-pg-edu-3:~#
```
*Добавляем к VM диск размером 5GB*

![image](https://github.com/gitsergeys/otus-pg-edu/assets/59079428/ab5f9a71-7edd-43b6-89be-969c5485b1ab)

*Запускаем VM, подключаемся и проверяем что диск появился рядом с основным*
```bash
script@otus-pg-edu-3:~$ sudo -i
root@otus-pg-edu-3:~# fdisk -l
Disk /dev/sda: 15 GiB, 16106127360 bytes, 31457280 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 416812CA-ED1B-4203-B926-990DB7A37190

Device       Start      End  Sectors  Size Type
/dev/sda1     2048     4095     2048    1M BIOS boot
/dev/sda2     4096  1054719  1050624  513M EFI System
/dev/sda3  1054720 31455231 30400512 14,5G Linux filesystem

Disk /dev/sdb: 5 GiB, 5368709120 bytes, 10485760 sectors
Disk model: VBOX HARDDISK
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```
*Приступаем к резметке, инициализпции диска и создания ФС*
```bash
root@otus-pg-edu-3:~# parted -l | grep Error
Error: /dev/sdb: unrecognised disk label
root@otus-pg-edu-3:~# lsblk
...
sda      8:0    0    15G  0 disk
├─sda1   8:1    0     1M  0 part
├─sda2   8:2    0   513M  0 part /boot/efi
└─sda3   8:3    0  14,5G  0 part /var/snap/firefox/common/host-hunspell
                                 /
sdb      8:16   0     5G  0 disk
...
root@otus-pg-edu-3:~# parted /dev/sdb mklabel gpt
root@otus-pg-edu-3:~# parted -a opt /dev/sdb mkpart primary ext4 0% 100%
root@otus-pg-edu-3:~# lsblk
...
sda      8:0    0    15G  0 disk
├─sda1   8:1    0     1M  0 part
├─sda2   8:2    0   513M  0 part /boot/efi
└─sda3   8:3    0  14,5G  0 part /var/snap/firefox/common/host-hunspell
                                 /
sdb      8:16   0     5G  0 disk
└─sdb1   8:17   0     5G  0 part
...
root@otus-pg-edu-3:~# mkfs.ext4 -L datapartition /dev/sdb1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 1310208 4k blocks and 327680 inodes
Filesystem UUID: 72c08943-5b15-43f0-8a75-8a2f143347df
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736

Allocating group tables: done
Writing inode tables: done
Creating journal (16384 blocks): done
Writing superblocks and filesystem accounting information: done
root@otus-pg-edu-3:~# e2label /dev/sdb1 otus_edu_pg
root@otus-pg-edu-3:~# lsblk --fs
NAME   FSTYPE   FSVER LABEL       UUID                                 FSAVAIL FSUSE% MOUNTPOINTS
sda
├─sda1
├─sda2 vfat     FAT32             C6D6-D777                             505,9M     1% /boot/efi
└─sda3 ext4     1.0               130d6b9b-80a2-4705-8b14-88c6c9b9c0f9    847M    89% /var/snap/firefox/common/host-hunspell
                                                                                      /
sdb
└─sdb1 ext4     1.0   otus_edu_pg 72c08943-5b15-43f0-8a75-8a2f143347df
sr0
root@otus-pg-edu-3:~# mkdir -p /mnt/data
root@otus-pg-edu-3:~# mount -o defaults /dev/sdb1 /mnt/data
root@otus-pg-edu-3:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           391M  1,5M  390M   1% /run
/dev/sda3        15G   13G  847M  94% /
tmpfs           2,0G  1,1M  2,0G   1% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
/dev/sda2       512M  6,1M  506M   2% /boot/efi
tmpfs           391M   92K  391M   1% /run/user/1000
/dev/sdb1       4,9G   24K  4,6G   1% /mnt/data
root@otus-pg-edu-3:~# vim /etc/fstab
UUID=72c08943-5b15-43f0-8a75-8a2f143347df /mnt/data ext4 defaults 0 2:
```
*Перезапускаем VM что-бы убедиться, что после перезапуска диск будет примонтирован*
```bash
root@otus-pg-edu-3:~# reboot
root@otus-pg-edu-3:~# df -h
Filesystem      Size  Used Avail Use% Mounted on
tmpfs           391M  6,0M  385M   2% /run
/dev/sda3        15G   13G  854M  94% /
tmpfs           2,0G     0  2,0G   0% /dev/shm
tmpfs           5,0M  4,0K  5,0M   1% /run/lock
/dev/sda2       512M  6,1M  506M   2% /boot/efi
/dev/sdb1       4,9G   28K  4,6G   1% /mnt/data
tmpfs           391M   68K  391M   1% /run/user/1000
```
**Все корректно**

*Делаем пользователя postgres владельцем*
```bash
root@otus-pg-edu-3:~# chown -R postgres:postgres /mnt/data/
```
*Переносим данные на новый диск и пытаемся запустить кластер*
```bash
root@otus-pg-edu-3:~# pg_ctlcluster 15 main stop
root@otus-pg-edu-3:~# mv /var/lib/postgresql/15 /mnt/data
root@otus-pg-edu-3:~# pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
**Видим что данных по пути нет, так как мы все перенесли на другой диск, поменялся путь**

*Ищем в каком конфиге прописан данный путь, что-бы его изменить и меняем на data_directory = '/mnt/data/15/main'*
```bash
root@otus-pg-edu-3:~# egrep -ir '/var/lib/postgresql/15/main' /etc/postgresql
/etc/postgresql/15/main/postgresql.conf:data_directory = '/var/lib/postgresql/15/main'          # use data in another directory
root@otus-pg-edu-3:~# cat /etc/postgresql/15/main/postgresql.conf
#data_directory = '/var/lib/postgresql/15/main' 
data_directory = '/mnt/data/15/main'
```
*Запускаем кластер и поверяем что все успешно, так как мы исправили путь на корректный*
```bash
root@otus-pg-edu-3:~# pg_ctlcluster 15 main start
root@otus-pg-edu-3:~# pg_ctlcluster 15 main status
pg_ctl: server is running (PID: 2017)
/usr/lib/postgresql/15/bin/postgres "-D" "/mnt/data/15/main" "-c" "config_file=/etc/postgresql/15/main/postgresql.conf"
```
*Подключаемся к psql и проверяем старые данные под postgres*
```bash
root@otus-pg-edu-3:~# sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | table_1 | table | postgres
(1 row)

postgres=# select * from table_1;
   test_info
----------------
 otus_education
(1 row)

postgres=#\q
```
**Все на месте**
****************
#### **Дополнительное задание со звездочкой**
-------------------------------------------------------
- 2 virtual machine (Ubuntu 22.04.3 LTS) oracle vm virtualbox
- 2 postgresql 15

*Подключаемся на новую VM*
```bash
script@otus-pg-edu-3-ext:~$ sudo -i
root@otus-pg-edu-3-ext:~#
```
*Устанавливаем PostgreSQL 15 и обновляемся*
```bash
root@otus-pg-edu-3-ext:~# apt update && apt upgrade -y -q && sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | apt-key add - && apt-get update && apt -y install postgresql-15
```
*Удаляем файлы с данными из /var/lib/postgres на новой/второй VM*
```bash
root@otus-pg-edu-3-ext:~# rm -f /var/lib/postgres/
```
*Процесс решения*
*Настроим экспорт директории на первом сервере, а затем выполним монтирование на втором сервере*
```bash
root@otus-pg-edu-3:~# pg_ctlcluster 15 main stop
root@otus-pg-edu-3:~# pg_ctlcluster 15 main status
pg_ctl: no server running
root@otus-pg-edu-3:~# apt install nfs-kernel-server -y
```
*На первом сервере добавим запись в файле /etc/exports для определения, указав путь с IP адресом второго сервера*
```bash
root@otus-pg-edu-3:~# vim /etc/exports
/mnt/data 192.168.1.83(rw,sync,no_subtree_check)
root@otus-pg-edu-3:~# sudo systemctl restart nfs-kernel-server
```
*Монтирование удаленной директории на втором сервере*
```bash
root@otus-pg-edu-3-ext:~# apt install nfs-common -y
root@otus-pg-edu-3-ext:~# mkdir -p /mnt/pg_external
root@otus-pg-edu-3-ext:~# chown -R postgres:postgres /mnt/pg_external
root@otus-pg-edu-3-ext:~# mount 192.168.1.180:/mnt/data /mnt/pg_external
root@otus-pg-edu-3-ext:~# vim /etc/fstab
192.168.1.180:/mnt/data /mnt/pg_external nfs defaults 0 2
root@otus-pg-edu-3-ext:~# df -h
Filesystem               Size  Used Avail Use% Mounted on
tmpfs                    391M  1,6M  390M   1% /run
/dev/sda3                 15G   14G   85M 100% /
tmpfs                    2,0G     0  2,0G   0% /dev/shm
tmpfs                    5,0M  4,0K  5,0M   1% /run/lock
/dev/sda2                512M  6,1M  506M   2% /boot/efi
tmpfs                    391M   92K  391M   1% /run/user/1000
192.168.1.180:/mnt/data  4,9G   39M  4,6G   1% /mnt/pg_external
```
*Редактируем конфиг postgresql.conf указав корректный путь и проверяем работу кластера с данными*
```bash
root@otus-pg-edu-3-ext:~# vim /etc/postgresql/15/main/postgresql.conf
data_directory = '/mnt/pg_external/15/main'
root@otus-pg-edu-3-ext:~# pg_ctlcluster 15 main start
root@otus-pg-edu-3-ext:~# sudo -u postgres psql
psql (15.5 (Ubuntu 15.5-1.pgdg22.04+1))
Type "help" for help.

postgres=# \dt
          List of relations
 Schema |  Name   | Type  |  Owner
--------+---------+-------+----------
 public | table_1 | table | postgres
(1 row)

postgres=# select * from table_1;
   test_info
----------------
 otus_education
(1 row)

postgres=#\q
```
**Все на месте**
