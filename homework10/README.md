# Домашнее задание #10

## Репликация



1. На 1 ВМ создаем таблицы test для записи, test2 для запросов на чтение.
```
CREATE DATABASE lab_repl1
CREATE ROLE user_repl1 WITH replication login password '1234567';
\c lab_repl1 
CREATE TABLE test(id integer primary key, name text);
CREATE TABLE test2(id integer primary key, name text);
GRANT SELECT ON TABLE test TO user_repl1;
INSERT INTO test (id, name) VALUES (1, 'ааа'), (2, 'ббб'), (3, 'ввв');
```
Отредактировала pg_hba.conf: добавила разрешение пользователю user_repl1 подключаться с ВМ #2 и ВМ #3 к БД lab_repl1.

2. Создаем публикацию таблицы test и подписываемся на публикацию таблицы test2 с ВМ №2.
```
ALTER SYSTEM SET wal_level = logical;

CREATE PUBLICATION test_pub FOR TABLE test;

CREATE SUBSCRIPTION test_sub
CONNECTION 'host=10.129.0.26 port=5432 user=user_repl2 dbname=lab_repl2 password=1234567'
PUBLICATION test_pub2;

SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16412
subname               | test_sub
pid                   | 6025
leader_pid            | 
relid                 | 
received_lsn          | 0/19635F0
last_msg_send_time    | 2023-12-27 20:39:57.963833+00
last_msg_receipt_time | 2023-12-27 20:39:57.966462+00
latest_end_lsn        | 0/19635F0
latest_end_time       | 2023-12-27 20:39:57.963833+00

```

3. На 2 ВМ создаем таблицы test2 для записи, test для запросов на чтение.
```
CREATE DATABASE lab_repl2;
CREATE ROLE user_repl2 WITH replication login password '1234567';
\c lab_repl2 
CREATE TABLE test(id integer primary key, name text);
CREATE TABLE test2(id integer primary key, name text);
GRANT SELECT ON TABLE test2 TO user_repl2;
```
Отредактировала pg_hba.conf: добавила разрешение пользователю user_repl2 подключаться с ВМ #1 и ВМ #3 к БД lab_repl2.

4. Создаем публикацию таблицы test2 и подписываемся на публикацию таблицы test1 с ВМ №1.
```
ALTER SYSTEM SET wal_level = logical;

CREATE PUBLICATION test_pub2 FOR TABLE test2;

CREATE SUBSCRIPTION test_sub2
CONNECTION 'host=10.129.0.15 port=5432 user=user_repl1 dbname=lab_repl1 password=1234567'
PUBLICATION test_pub;

SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16410
subname               | test_sub2
pid                   | 5950
leader_pid            | 
relid                 | 
received_lsn          | 0/1994E90
last_msg_send_time    | 2023-12-27 20:39:50.992427+00
last_msg_receipt_time | 2023-12-27 20:39:50.990188+00
latest_end_lsn        | 0/1994E90
latest_end_time       | 2023-12-27 20:39:50.992427+00
```

5. 3 ВМ использовать как реплику для чтения и бэкапов (подписаться на таблицы из ВМ №1 и №2 ).
```
CREATE DATABASE lab_repl3;
\c lab_repl3 
CREATE TABLE test(id integer primary key, name text);
CREATE TABLE test2(id integer primary key, name text);

CREATE SUBSCRIPTION test_sub3
CONNECTION 'host=10.129.0.26 port=5432 user=user_repl2 dbname=lab_repl2 password=1234567'
PUBLICATION test_pub2;

CREATE SUBSCRIPTION test_sub3_2
CONNECTION 'host=10.129.0.15 port=5432 user=user_repl1 dbname=lab_repl1 password=1234567'
PUBLICATION test_pub;

SELECT * FROM pg_stat_subscription \gx
-[ RECORD 1 ]---------+------------------------------
subid                 | 16419
subname               | test_sub3
pid                   | 5315
leader_pid            | 
relid                 | 
received_lsn          | 0/1963AB8
last_msg_send_time    | 2023-12-27 20:45:06.66207+00
last_msg_receipt_time | 2023-12-27 20:45:06.660341+00
latest_end_lsn        | 0/1963AB8
latest_end_time       | 2023-12-27 20:45:06.66207+00
-[ RECORD 2 ]---------+------------------------------
subid                 | 16420
subname               | test_sub3_2
pid                   | 5319
leader_pid            | 
relid                 | 
received_lsn          | 0/1995360
last_msg_send_time    | 2023-12-27 20:45:06.664235+00
last_msg_receipt_time | 2023-12-27 20:45:06.660308+00
latest_end_lsn        | 0/1995360
latest_end_time       | 2023-12-27 20:45:06.664235+00
```

Проверила: изменения в таблицах реплицируются в соответствии со сделанными настройками.

### Задание со *

Реализовать горячее реплицирование для высокой доступности на 4ВМ. Источником должна выступать ВМ №3. Написать с какими проблемами столкнулись.

Выполнила настройки:
 
Настройки в postgresql.conf менять не нужно - по умолчанию:
wal_level = replica
hot_standby = on

#### На ВМ #3: 
* В pg_hba.conf добавила настройку для разрешения подключения репликации с ВМ #4.
* Сменила пароль пользователя postgres (потребуется при настройке репликации)
* Создала слот для репликации:
```
SELECT pg_create_physical_replication_slot('replica_hot');
```
Слот репликации нужен, так как не используется архив журнала предзаписи, и при определенной задержке мастер может успеть удалить необходимые сегменты. 

#### На ВМ #4:
* Остановила PostgreSQL и удалила все файлы в каталоге с данными:
```
rm -rf /var/lib/postgresql/16/main/*
```
* Выполнила копирование данных с помощью pg_basebackup с ключом -R, который указывает, что нужно  создать файл standby.signal и добавить параметры конфигурации в файл postgresql.auto.conf в целевом каталоге
```
pg_basebackup --pgdata=/var/lib/postgresql/16/main -R --slot=replica_hot -h 10.129.0.25 -p 5432 -U postgres -X stream
```
*  Проверила, что был автоматически создан файл /var/lib/postgresql/16/main/postgresql.auto.conf, в котором указаны параметры соединения с мастером и имя слота репликации, и файл standby.signal
* Запустила Postgres, проверила что есть все данные с ВМ #3

Проверила, что при изменении данных в опубликованных таблицах на ВМ #1 и #2 данные в таблицах, где настроена подписка на всех трёх ВМ, также изменяются. Изменения с ВМ #3 реплицируются на ВМ #4.
