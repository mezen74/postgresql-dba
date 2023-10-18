# Домашнее задание #3

## Установка и настройка PostgreSQL

- создайте виртуальную машину c Ubuntu 20.04/22.04 LTS в GCE/ЯО/Virtual Box/докере
**Работу выполняла в Яндекс-Облаке**
- поставьте на нее PostgreSQL 15 через sudo apt
```
sudo apt update && sudo apt upgrade -y && sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list' && wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add - && sudo apt-get update && sudo apt-get -y install postgresql-15
```
- проверьте что кластер запущен
```
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory              Log file
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
```
- зайдите из под пользователя postgres в psql и сделайте произвольную таблицу с произвольным содержимым
```
sudo -u postgres psql

postgres=# create table test(c1 text);
CREATE TABLE
postgres=# insert into test values('1');
INSERT 0 1
```
- остановите postgres
```
sudo -u postgres pg_ctlcluster 15 main stop
```
- создайте новый диск к ВМ размером 10GB
- добавьте свеже-созданный диск к виртуальной машине - надо зайти в режим ее редактирования и дальше выбрать пункт attach existing disk
- проинициализируйте диск согласно инструкции и подмонтировать файловую систему, только не забывайте менять имя диска на актуальное, в вашем случае это скорее всего будет /dev/sdb - https://www.digitalocean.com/community/tutorials/how-to-partition-and-format-storage-devices-in-linux
```
sudo parted /dev/vda2 mklabel gpt
sudo parted -a opt /dev/vdb mkpart primary ext4 0% 100%
sudo mkfs.ext4 -L datapartition /dev/vdb1
sudo mkdir -p /mnt/data
```
добавила в /etc/fstab:
```
LABEL=datapartition /mnt/data ext4 defaults 0 2
```
- перезагрузите инстанс и убедитесь, что диск остается примонтированным (если не так смотрим в сторону fstab)
```
df -h -x tmpfs
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda2        18G  4.5G   13G  27% /
/dev/vdb1       9.8G   24K  9.3G   1% /mnt/data
```

- сделайте пользователя postgres владельцем /mnt/data
```
sudo chown -R postgres:postgres /mnt/data/
```
- перенесите содержимое /var/lib/postgres/15 в /mnt/data
```
sudo -u postgres mv /var/lib/postgresql/15 /mnt/data
```
- попытайтесь запустить кластер
```
sudo -u postgres pg_ctlcluster 15 main start
Error: /var/lib/postgresql/15/main is not accessible or does not exist
```
- напишите получилось или нет и почему

**Кластер не запустился (сообщение об ошибке см.выше), потому что каталог *main*, в котором хранятся данные, отсутствует в */var/lib/postgresql/15/* - я перенесла его в другое место на предыдущем шаге, а в конфигурационном файле путь к каталогу с данными не поменяла.**
- задание: найти конфигурационный параметр в файлах раположенных в /etc/postgresql/15/main который надо поменять и поменяйте его.

**Поменяла файл */etc/postgresql/15/main/postgresql.conf* (предварительно сохранив оригинальный), проверила изменения:**
```
diff -U2 /etc/postgresql/15/main/postgresql.conf.orig /etc/postgresql/15/main/postgresql.conf
--- /etc/postgresql/15/main/postgresql.conf.orig	2023-10-18 16:50:56.197025384 +0000
+++ /etc/postgresql/15/main/postgresql.conf	2023-10-18 16:52:44.054979193 +0000
@@ -40,5 +40,5 @@
 # option or PGDATA environment variable, represented here as ConfigDir.
 
-data_directory = '/var/lib/postgresql/15/main'		# use data in another directory
+data_directory = '/mnt/data/15/main'		# use data in another directory
```

- напишите что и почему поменяли

**Поменяла параметр data_directory: указала путь к каталогу, в который ранее переместила файлы**
- попытайтесь запустить кластер
```
sudo -u postgres pg_ctlcluster 15 main start
sudo -u postgres pg_lsclusters
Ver Cluster Port Status Owner    Data directory    Log file
15  main    5432 online postgres /mnt/data/15/main /var/log/postgresql/postgresql-15-main.log

```
- напишите получилось или нет и почему

**Получилось, кластер запустился, потому что в файле конфигурации путь к каталогу с данными теперь указан корректно.**

- зайдите через через psql и проверьте содержимое ранее созданной таблицы

**Таблица есть, данные есть**
```
sudo -u postgres psql

postgres=# select * from test;
 c1 
----
 1
(1 row)
```


## Задание со звездочкой *

Не удаляя существующий инстанс ВМ сделайте новый, поставьте на него PostgreSQL, удалите файлы с данными из /var/lib/postgres, перемонтируйте внешний диск который сделали ранее от первой виртуальной машины ко второй и запустите PostgreSQL на второй машине так чтобы он работал с данными на внешнем диске, расскажите как вы это сделали и что в итоге получилось.

- Создала ВМ Centos 7
- Установила PostgreSQL 15. Понадобилось также подключить репозиторий EPEL, иначе PostgreSQL не устанавливался из-за того, что не мог разрешить зависимость для библиотеки *libzstd.so.1*
```
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

sudo yum install -y postgresql15-server
```
- Инициализировала и запустила кластер :
```
sudo /usr/pgsql-15/bin/postgresql-15-setup initdb
sudo systemctl enable postgresql-15
sudo systemctl start postgresql-15
```
- Проверила, что кластер запущен, подключилась к БД postgres.
- Остановила кластер. Файлы в /var/lib/pgsql/ не удаляла (на всякий случай).
- Подключила диск от первой ВМ. Для этого сначала пришлось отсоединить его от первой ВМ, иначе его не было в списке дисков, доступных для подключения.
- Создала каталог /mnt/data и настроила монтирование в него подключенного диска - отредактировала файл /etc/fstab аналогично первой ВМ.
- Сделала пользователя postgres владельцем /mnt/data/. Если этого не сделать, то Postgres не запустится из-за отсутствия прав доступа.
```
sudo chown -R postgres:postgres /mnt/data/
```
- Сменила контекст SELinux для /mnt/data:
```
chcon -Rt postgresql_db_t /mnt/data
```
- Переопределила переменную PGDATA для сервиса postgresql-15:
```
sudo systemctl edit postgresql-15
```
В появившемся окне указала следующие параметры:
```
[Service]
Environment=PGDATA=/mnt/data/15/main/
```
- Запустила кластер, получила ошибку:
```
Oct 18 19:53:37 hw03-centos.ru-central1.internal postmaster[8316]: postmaster: could not access the server configuration file "/mnt/data/15/main/postgresql.conf": No such file or directory
```
Этого файла там действительно нет, на Ubuntu  он хранится в другом месте
- Скопировала файл *postgresql.conf* из */var/lib/pgsql/15/data/*, также скопировала и *pg_hba.conf* и *pg_ident.conf*. Файлы конфигурации типовые, ничего в них не меняла.
```
sudo -u postgres cp /var/lib/pgsql/15/data/postgresql.conf /mnt/data/15/main/
sudo -u postgres cp /var/lib/pgsql/15/data/pg_hba.conf /mnt/data/15/main/
sudo -u postgres cp /var/lib/pgsql/15/data/pg_ident.conf /mnt/data/15/main/
```
- Запустила кластер, на этот раз успешно.
- Зашла через *psql* и проверила, что созданная на первой машине таблица есть, данные в ней есть:
```
sudo -u postgres psql
postgres=# select * from test;
 c1 
----
 1
(1 row)

```
- Удалила все файлы в */var/lib/pgsql*
```
sudo rm -rf /var/lib/pgsql/*
```
- Перезапустила кластер, убедилась что кластер запустился успешно. 
- Зашла через psql и ещё раз проверила, что созданная на первой машине таблица есть, данные в ней есть

