# Тесты

## Штатные процедуры

### Переключение мастера (switchover)

Проверяем текущее состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 10 |           |
| pg-db2 | 192.168.2.12 | Replica | streaming | 10 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```

Выполняем переключение:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml switchover
Current cluster topology
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 10 |           |
| pg-db2 | 192.168.2.12 | Replica | streaming | 10 |         0 |
+--------+--------------+---------+-----------+----+-----------+
Primary [pg-db1]: 
Candidate ['pg-db2'] []: 
When should the switchover take place (e.g. 2024-02-07T19:50 )  [now]: 
Are you sure you want to switchover cluster pg_cluster, demoting current leader pg-db1? [y/N]: y
2024-02-07 18:50:54.94738 Successfully switched over to "pg-db2"

+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | stopped |    |   unknown |
| pg-db2 | 192.168.2.12 | Leader  | running | 10 |           |
+--------+--------------+---------+---------+----+-----------+
```

Снова проверяем состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 11 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 11 |           |
+--------+--------------+---------+-----------+----+-----------+

```
**Было выполнено переключение мастера с хоста pg-db1 на pg-db2**


### Failover штатными средствами patroni

Проверяем состояние кластера:
```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 18 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 18 |           |
| pg-db3 | 192.168.2.13 | Replica | streaming | 18 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```

Выполняем команду failover:
```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml failover
Current cluster topology
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 18 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 18 |           |
| pg-db3 | 192.168.2.13 | Replica | streaming | 18 |         0 |
+--------+--------------+---------+-----------+----+-----------+
Candidate ['pg-db1', 'pg-db3'] []: pg-db1
Are you sure you want to failover cluster pg_cluster, demoting current leader pg-db2? [y/N]: y
2024-02-10 17:07:55.95880 Successfully failed over to "pg-db1"
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 18 |           |
| pg-db2 | 192.168.2.12 | Replica | stopped   |    |   unknown |
| pg-db3 | 192.168.2.13 | Replica | streaming | 18 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```
Снова проверяем состояние кластера:
```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 19 |           |
| pg-db2 | 192.168.2.12 | Replica | streaming | 19 |         0 |
| pg-db3 | 192.168.2.13 | Replica | streaming | 19 |         0 |
+--------+--------------+---------+-----------+----+-----------+

```
**Было выполнено переключение мастера с хоста pg-db1 на pg-db2**

### Изменение конфигурации Postgres средствами Patroni
```
patronictl -c /etc/patroni/patroni.yml edit-config

[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml edit-config
--- 
+++ 
@@ -4,7 +4,18 @@
 loop_wait: 10
 maximum_lag_on_failover: 1048576
 postgresql:
-  parameters: null
+  parameters:
+    shared_buffers: 1GB
+    effective_cache_size: 3GB
+    work_mem: 8MB
+    maintenance_work_mem: 256MB
+    random_page_cost: 1.1
+    effective_io_concurrency: 200
+    checkpoint_completion_target: 0.9
+    wal_buffers: 16MB
+    min_wal_size: 4GB
+    max_wal_size: 16GB
+    synchronous_commit: off
   pg_hba:
   - host replication replicator 127.0.0.1/32 md5
   - host replication replicator 192.168.2.0/24 md5

Apply these changes? [y/N]: y
Configuration changed
```

Записи из логов на обоих серверах:
```
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed effective_cache_size from 524288 to 3GB
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed effective_io_concurrency from 1 to 200
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed maintenance_work_mem from 65536 to 256MB
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed max_wal_size from 1024 to 16GB
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed min_wal_size from 80 to 4GB
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed random_page_cost from 4 to 1.1
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed shared_buffers from 16384 to 1GB (restart might be required)
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed synchronous_commit from on to False
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,093 INFO: Changed wal_buffers from 512 to 16MB (restart might be required)
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,094 INFO: Changed work_mem from 4096 to 8MB
Feb  7 18:19:27 pg-db1 patroni: 2024-02-07 18:19:27,099 INFO: Reloading PostgreSQL configuration.
Feb  7 18:19:27 pg-db1 patroni: server signaled


Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed effective_cache_size from 524288 to 3GB
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed effective_io_concurrency from 1 to 200
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed maintenance_work_mem from 65536 to 256MB
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed max_wal_size from 1024 to 16GB
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed min_wal_size from 80 to 4GB
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,106 INFO: Changed random_page_cost from 4 to 1.1
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,107 INFO: Changed shared_buffers from 16384 to 1GB (restart might be required)
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,107 INFO: Changed synchronous_commit from on to False
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,107 INFO: Changed wal_buffers from 512 to 16MB (restart might be required)
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,107 INFO: Changed work_mem from 4096 to 8MB
Feb  7 18:19:27 pg-db2 patroni: 2024-02-07 18:19:27,111 INFO: Reloading PostgreSQL configuration.
Feb  7 18:19:27 pg-db2 patroni: server signaled
```

В логах видно, что для применения некоторых параметров требуется перезапустить PostgreSQL.

Это же видно при проверке состояния кластера: в столбце 'Pending' для обоих серверов стоит символ 'звёздочка'.


```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+-----------------+
| Member | Host         | Role    | State     | TL | Lag in MB | Pending restart |
+--------+--------------+---------+-----------+----+-----------+-----------------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 10 |           | *               |
| pg-db2 | 192.168.2.12 | Replica | streaming | 10 |         0 | *               |
+--------+--------------+---------+-----------+----+-----------+-----------------+
```
Patroni в этом случае не выполняет перезапуск PostgreSQL, это действие должен выполнить администратор.


Выполняем рестарт PostgreSQL поочередно на обоих серверах:
```
patronictl -c /etc/patroni/patroni.yml restart pg_cluster pg-db2
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+-----------------+
| Member | Host         | Role    | State     | TL | Lag in MB | Pending restart |
+--------+--------------+---------+-----------+----+-----------+-----------------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 10 |           | *               |
| pg-db2 | 192.168.2.12 | Replica | streaming | 10 |         0 | *               |
+--------+--------------+---------+-----------+----+-----------+-----------------+
When should the restart take place (e.g. 2024-02-07T19:29)  [now]: 
Are you sure you want to restart members pg-db2? [y/N]: y
Restart if the PostgreSQL version is less than provided (e.g. 9.5.2)  []: 
Success: restart on member pg-db2

patronictl -c /etc/patroni/patroni.yml restart pg_cluster pg-db1
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+-----------------+
| Member | Host         | Role    | State     | TL | Lag in MB | Pending restart |
+--------+--------------+---------+-----------+----+-----------+-----------------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 10 |           | *               |
| pg-db2 | 192.168.2.12 | Replica | streaming | 10 |         0 |                 |
+--------+--------------+---------+-----------+----+-----------+-----------------+
When should the restart take place (e.g. 2024-02-07T19:34)  [now]: 
Are you sure you want to restart members pg-db1? [y/N]: y
Restart if the PostgreSQL version is less than provided (e.g. 9.5.2)  []: 
Success: restart on member pg-db1
```

Проверяем состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running | 10 |           |
| pg-db2 | 192.168.2.12 | Replica | running | 10 |         0 |
+--------+--------------+---------+---------+----+-----------+
```
Перезапуск PostgreSQL больше не требуется.

Проверяем значение параметров PostgreSQL - подключаемся с помощью psql и выборочно проверяем значение некоторых параметров:
```
[yc-user@pg-db1 ~]$ psql -h 192.168.2.10 -p 5000 -U postgres
Password for user postgres: 
psql (15.5)
Type "help" for help.

postgres=# SELECT name, setting, unit,
          boot_val, reset_val,
          source, sourcefile, sourceline,
          pending_restart, context
   FROM   pg_settings WHERE name = 'maintenance_work_mem'\gx
-[ RECORD 1 ]---+---------------------------------------
name            | maintenance_work_mem
setting         | 262144
unit            | kB
boot_val        | 65536
reset_val       | 262144
source          | configuration file
sourcefile      | /var/lib/pgsql/15/data/postgresql.conf
sourceline      | 11
pending_restart | f
context         | user

postgres=# SELECT name, setting, unit,
          boot_val, reset_val,
          source, sourcefile, sourceline,
          pending_restart, context
   FROM   pg_settings WHERE name = 'work_mem'\gx
-[ RECORD 1 ]---+---------------------------------------
name            | work_mem
setting         | 8192
unit            | kB
boot_val        | 4096
reset_val       | 8192
source          | configuration file
sourcefile      | /var/lib/pgsql/15/data/postgresql.conf
sourceline      | 30
pending_restart | f
context         | user


[yc-user@pg-db1 ~]$ psql -h 192.168.2.10 -p 5001 -U postgres
Password for user postgres: 
psql (15.5)
Type "help" for help.

postgres=# SELECT name, setting, unit,
postgres-#           boot_val, reset_val,
postgres-#           source, sourcefile, sourceline,
postgres-#           pending_restart, context
postgres-#    FROM   pg_settings WHERE name = 'maintenance_work_mem'\gx
-[ RECORD 1 ]---+---------------------------------------
name            | maintenance_work_mem
setting         | 262144
unit            | kB
boot_val        | 65536
reset_val       | 262144
source          | configuration file
sourcefile      | /var/lib/pgsql/15/data/postgresql.conf
sourceline      | 11
pending_restart | f
context         | user

postgres=# SELECT name, setting, unit,
          boot_val, reset_val,
          source, sourcefile, sourceline,
          pending_restart, context
   FROM   pg_settings WHERE name = 'work_mem'\gx
-[ RECORD 1 ]---+---------------------------------------
name            | work_mem
setting         | 8192
unit            | kB
boot_val        | 4096
reset_val       | 8192
source          | configuration file
sourcefile      | /var/lib/pgsql/15/data/postgresql.conf
sourceline      | 30
pending_restart | f
context         | user
```

Можно посмотреть текущую конфигурацию средствами Patroni:
```
patronictl -c /etc/patroni/patroni.yml show-config
initdb:
- encoding: UTF8
- data-checksums
loop_wait: 10
maximum_lag_on_failover: 1048576
postgresql:
  parameters:
    checkpoint_completion_target: 0.9
    effective_cache_size: 3GB
    effective_io_concurrency: 200
    maintenance_work_mem: 256MB
    max_wal_size: 16GB
    min_wal_size: 4GB
    random_page_cost: 1.1
    shared_buffers: 1GB
    synchronous_commit: false
    wal_buffers: 16MB
    work_mem: 8MB
  pg_hba:
  - host replication replicator 127.0.0.1/32 md5
  - host replication replicator 192.168.2.0/24 md5
  - host all all 0.0.0.0/0 md5
  use_pg_rewind: true
retry_timeout: 10
ttl: 30
users:
  options:
  - createrole
  - createdb
  password: otus135
  pgadmin: null
```
**Конфигурация изменена на обоих серверах, новые параметры применены**


### Штатная остановка сервиса Patroni

Останавливаем сервис на мастере:
```
[yc-user@pg-db2 ~]$ sudo systemctl stop patroni
```
Проверяем состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running | 12 |           |
| pg-db2 | 192.168.2.12 | Replica | stopped |    |   unknown |
+--------+--------------+---------+---------+----+-----------+

[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) ----+-----------+
| Member | Host         | Role   | State   | TL | Lag in MB |
+--------+--------------+--------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader | running | 12 |           |
+--------+--------------+--------+---------+----+-----------+
```
Записи в логе сервера:
```
Feb 11 15:16:16 pg-db2 systemd: Stopping Runners to orchestrate a high-availability PostgreSQL...
Feb 11 15:16:17 pg-db2 systemd: Stopped Runners to orchestrate a high-availability PostgreSQL.
```
**Перед остановкой Patroni штатно остановил PostgreSQL**

Снова запускаем patroni:
```
[yc-user@pg-db2 ~]$ sudo systemctl start patroni
```
Проверяем состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running | 12 |           |
| pg-db2 | 192.168.2.12 | Replica | running | 11 |         0 |
+--------+--------------+---------+---------+----+-----------+
```
**Штатная остановка Patroni не вызывает проблем в работе кластера. Прежде чем завершить работу, Patroni останавливает PostgreSQL**

### Завершение работы сервера:

На хосте pg-db1 выполняем команду:
```
sudo systemctl poweroff
```
Проверяем состояние кластера:

```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | stopped |    |   unknown |
| pg-db2 | 192.168.2.12 | Leader  | running | 18 |           |
+--------+--------------+---------+---------+----+-----------+

[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) ----+-----------+
| Member | Host         | Role   | State   | TL | Lag in MB |
+--------+--------------+--------+---------+----+-----------+
| pg-db2 | 192.168.2.12 | Leader | running | 18 |           |
+--------+--------------+--------+---------+----+-----------+
```
**Patroni перевёл роль мастера на хост pg-db2 перед завершением работы сервера**

После включения сервера:
```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 18 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 18 |           |
+--------+--------------+---------+-----------+----+-----------+
```

### Добавление нового узла в кластер

Развернуть ВМ, установить PostgreSQL и Patroni (как описано в документе "Установка").
В Prometheus добавить адрес нового узла, чтобы данные с него также отображались в мониторинге.

После запуска Patroni на новом узле проверяем состояние кластера:
```
[yc-user@pg-db2 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) ---------+----+-----------+
| Member | Host         | Role    | State            | TL | Lag in MB |
+--------+--------------+---------+------------------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming        | 18 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running          | 18 |           |
| pg-db3 | 192.168.2.13 | Replica | streaming        | 18 |         0 |
+--------+--------------+---------+------------------+----+-----------+
```
Записи из логов нового узла:
```
Feb 10 16:55:10 pg-db3 systemd: Started Runners to orchestrate a high-availability PostgreSQL.
Feb 10 16:55:10 pg-db3 patroni: 2024-02-10 16:55:10,879 INFO: Selected new etcd server http://pg-proxy:2379
Feb 10 16:55:10 pg-db3 patroni: 2024-02-10 16:55:10,908 INFO: No PostgreSQL configuration items changed, nothing to reload.
Feb 10 16:55:10 pg-db3 patroni: 2024-02-10 16:55:10,916 INFO: Lock owner: pg-db2; I am pg-db3
Feb 10 16:55:10 pg-db3 patroni: 2024-02-10 16:55:10,917 INFO: trying to bootstrap from leader 'pg-db2'
Feb 10 16:55:20 pg-db3 patroni: 2024-02-10 16:55:20,915 INFO: Lock owner: pg-db2; I am pg-db3
Feb 10 16:55:20 pg-db3 patroni: 2024-02-10 16:55:20,917 INFO: bootstrap from leader 'pg-db2' in progress
Feb 10 16:55:30 pg-db3 patroni: 2024-02-10 16:55:30,914 INFO: Lock owner: pg-db2; I am pg-db3
Feb 10 16:55:30 pg-db3 patroni: 2024-02-10 16:55:30,916 INFO: bootstrap from leader 'pg-db2' in progress
Feb 10 16:55:40 pg-db3 patroni: 2024-02-10 16:55:40,915 INFO: Lock owner: pg-db2; I am pg-db3
Feb 10 16:55:40 pg-db3 patroni: 2024-02-10 16:55:40,917 INFO: bootstrap from leader 'pg-db2' in progress
Feb 10 16:55:42 pg-db3 patroni: 2024-02-10 16:55:42,416 INFO: replica has been created using basebackup
Feb 10 16:55:42 pg-db3 patroni: 2024-02-10 16:55:42,417 INFO: bootstrapped from leader 'pg-db2'
Feb 10 16:55:43 pg-db3 patroni: 2024-02-10 16:55:43,018 INFO: postmaster pid=8319
Feb 10 16:55:43 pg-db3 patroni: 2024-02-10 16:55:43.028 UTC [8319] LOG:  redirecting log output to logging collector process
Feb 10 16:55:43 pg-db3 patroni: 2024-02-10 16:55:43.028 UTC [8319] HINT:  Future log output will appear in directory "log".
Feb 10 16:55:43 pg-db3 patroni: localhost:5432 - rejecting connections
Feb 10 16:55:43 pg-db3 patroni: localhost:5432 - rejecting connections
Feb 10 16:55:44 pg-db3 patroni: localhost:5432 - accepting connections
Feb 10 16:55:44 pg-db3 patroni: 2024-02-10 16:55:44,079 INFO: Lock owner: pg-db2; I am pg-db3
Feb 10 16:55:44 pg-db3 patroni: 2024-02-10 16:55:44,080 INFO: establishing a new patroni heartbeat connection to postgres
Feb 10 16:55:44 pg-db3 patroni: 2024-02-10 16:55:44,103 INFO: no action. I am (pg-db3), a secondary, and following a leader (pg-db2)
```
Убедиться, что на новом узле применены параметры конфигурации PostgreSQL, общие для всего кластера.


## Тесты отказов сервисов

### Прерывание работы Postgres

Прерываем работу PostgreSQL с помощью сигнала SIGKILL (немедленно прекращает работу, невозможно игнорировать)

```
sudo yum install psmisc
sudo killall -KILL postgres
```

Из логов системы:
```
Feb 10 16:04:10 pg-db1 patroni: patroni.exceptions.PostgresConnectionException: connection problems
Feb 10 16:04:11 pg-db1 patroni: 2024-02-10 16:04:11,724 WARNING: Postgresql is not running.
Feb 10 16:04:11 pg-db1 patroni: 2024-02-10 16:04:11,724 INFO: Lock owner: pg-db1; I am pg-db1
Feb 10 16:04:11 pg-db1 patroni: 2024-02-10 16:04:11,739 INFO: pg_controldata:
Feb 10 16:04:11 pg-db1 patroni: pg_control version number: 1300
Feb 10 16:04:11 pg-db1 patroni: Catalog version number: 202209061
Feb 10 16:04:11 pg-db1 patroni: Database system identifier: 7331748476172603812
Feb 10 16:04:11 pg-db1 patroni: Database cluster state: in production
Feb 10 16:04:11 pg-db1 patroni: pg_control last modified: Sat Feb 10 15:56:22 2024
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint location: 0/921889F8
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's REDO location: 0/7CE78998
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's REDO WAL file: 0000000F000000000000007C
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's TimeLineID: 15
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's PrevTimeLineID: 15
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's full_page_writes: on
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's NextXID: 0:6286
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's NextOID: 24590
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's NextMultiXactId: 1
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's NextMultiOffset: 0
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestXID: 717
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestXID's DB: 1
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestActiveXID: 6285
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestMultiXid: 1
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestMulti's DB: 1
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's oldestCommitTsXid: 0
Feb 10 16:04:11 pg-db1 patroni: Latest checkpoint's newestCommitTsXid: 0
Feb 10 16:04:11 pg-db1 patroni: Time of latest checkpoint: Sat Feb 10 15:51:52 2024
Feb 10 16:04:11 pg-db1 patroni: Fake LSN counter for unlogged rels: 0/3E8
Feb 10 16:04:11 pg-db1 patroni: Minimum recovery ending location: 0/0
Feb 10 16:04:11 pg-db1 patroni: Min recovery ending loc's timeline: 0
Feb 10 16:04:11 pg-db1 patroni: Backup start location: 0/0
Feb 10 16:04:11 pg-db1 patroni: Backup end location: 0/0
Feb 10 16:04:11 pg-db1 patroni: End-of-backup record required: no
Feb 10 16:04:11 pg-db1 patroni: wal_level setting: replica
Feb 10 16:04:11 pg-db1 patroni: wal_log_hints setting: on
Feb 10 16:04:11 pg-db1 patroni: max_connections setting: 100
Feb 10 16:04:11 pg-db1 patroni: max_worker_processes setting: 8
Feb 10 16:04:11 pg-db1 patroni: max_wal_senders setting: 10
Feb 10 16:04:11 pg-db1 patroni: max_prepared_xacts setting: 0
Feb 10 16:04:11 pg-db1 patroni: max_locks_per_xact setting: 64
Feb 10 16:04:11 pg-db1 patroni: track_commit_timestamp setting: off
Feb 10 16:04:11 pg-db1 patroni: Maximum data alignment: 8
Feb 10 16:04:11 pg-db1 patroni: Database block size: 8192
Feb 10 16:04:11 pg-db1 patroni: Blocks per segment of large relation: 131072
Feb 10 16:04:11 pg-db1 patroni: WAL block size: 8192
Feb 10 16:04:11 pg-db1 patroni: Bytes per WAL segment: 16777216
Feb 10 16:04:11 pg-db1 patroni: Maximum length of identifiers: 64
Feb 10 16:04:11 pg-db1 patroni: Maximum columns in an index: 32
Feb 10 16:04:11 pg-db1 patroni: Maximum size of a TOAST chunk: 1996
Feb 10 16:04:11 pg-db1 patroni: Size of a large-object chunk: 2048
Feb 10 16:04:11 pg-db1 patroni: Date/time type storage: 64-bit integers
Feb 10 16:04:12 pg-db1 patroni: Float8 argument passing: by value
Feb 10 16:04:12 pg-db1 patroni: Data page checksum version: 0
Feb 10 16:04:12 pg-db1 patroni: Mock authentication nonce: feb9b41a9a1e27c5c012c8054011aabdd0f544288ecae425c06cfb1b7d88396e
Feb 10 16:04:12 pg-db1 patroni: 2024-02-10 16:04:11,743 INFO: starting primary after failure
```
**Patroni обнаружил что PostgreSQL не запущен и запустил его.**

Проверяем состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 15 |           |
| pg-db2 | 192.168.2.12 | Replica | streaming | 15 |         0 |
+--------+--------------+---------+-----------+----+-----------+
```
**Мастер не изменился.**

**Patroni отслеживает состояние PostgreSQL и в случае необходимости запускает его.**

### Приостановка работы PostgreSQL

Отправляем PostgreSQL сигнал SIGSTOP (немедленно приостанавливает работу не завершаясь, невозможно игнорировать)

На ноде pg-db1 (на которой запущен мастер) выполняем команду:
```
sudo killall -STOP postgres
```
Мастер сменился, в логах - ничего нет. Выглядит так, что работу приостановил не только PostgreSQL, но и Patroni

```
patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) ----+-----------+
| Member | Host         | Role   | State   | TL | Lag in MB |
+--------+--------------+--------+---------+----+-----------+
| pg-db2 | 192.168.2.11 | Leader | running | 16 |           |
+--------+--------------+--------+---------+----+-----------+
```

В логах на pg-db2:
```
Feb 11 15:33:44 pg-db1 patroni: 2024-02-11 15:33:44,546 WARNING: Request failed to pg-db2: GET http://192.168.2.12:8008/patroni (HTTPConnectionPool(host='192.168.2.12', port=8008): Max retri
es exceeded with url: /patroni (Caused by ReadTimeoutError("HTTPConnectionPool(host='192.168.2.12', port=8008): Read timed out. (read timeout=2)",)))
Feb 11 15:33:44 pg-db1 patroni: 2024-02-11 15:33:44,554 INFO: promoted self to leader by acquiring session lock
Feb 11 15:33:44 pg-db1 patroni: server promoting
Feb 11 15:33:46 pg-db1 patroni: 2024-02-11 15:33:46,591 INFO: no action. I am (pg-db1), the leader with the lock
```
**На pg-db2 Patroni обнаружил недоступность второго узла и поднял реплику до мастера.**

Возобновим работу PostgreSQL на pg-db1 путём отправки ему сигнала SIGCONT:
```
sudo killall -CONT postgres
```
Проверим состояние кластера:
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 17 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 17 |           |
+--------+--------------+---------+-----------+----+-----------+
```
Видим, что на обоих серверах PostgreSQL теперь запущен, на pg-db2 реплика.

Записи в логах:
```
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,203 INFO: demoted self because failed to update leader lock in DCS
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,205 INFO: closed patroni connections to postgres
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,208 WARNING: Loop time exceeded, rescheduling immediately.
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,217 INFO: Lock owner: pg-db1; I am pg-db2
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,219 INFO: starting after demotion in progress
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53,820 INFO: postmaster pid=3599
Feb 11 15:34:53 pg-db2 patroni: localhost:5432 - no response
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53.842 UTC [3599] LOG:  redirecting log output to logging collector process
Feb 11 15:34:53 pg-db2 patroni: 2024-02-11 15:34:53.842 UTC [3599] HINT:  Future log output will appear in directory "log".
Feb 11 15:34:54 pg-db2 patroni: localhost:5432 - accepting connections
Feb 11 15:34:54 pg-db2 patroni: localhost:5432 - accepting connections
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,861 INFO: Lock owner: pg-db1; I am pg-db2
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,861 INFO: establishing a new patroni heartbeat connection to postgres
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,885 INFO: Local timeline=23 lsn=0/A75AE0C8
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,905 INFO: primary_timeline=24
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,906 INFO: primary: history=20#0110/A75A3DC0#011no recovery target specified
Feb 11 15:34:54 pg-db2 patroni: 21#0110/A75A9A88#011no recovery target specified
Feb 11 15:34:54 pg-db2 patroni: 22#0110/A75ADEE8#011no recovery target specified
Feb 11 15:34:54 pg-db2 patroni: 23#0110/A75AE0C8#011no recovery target specified
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,923 INFO: Dropped unknown replication slot 'pg_db1'
Feb 11 15:34:54 pg-db2 patroni: 2024-02-11 15:34:54,935 INFO: no action. I am (pg-db2), a secondary, and following a leader (pg-db1)
Feb 11 15:34:55 pg-db2 patroni: 2024-02-11 15:34:55,456 INFO: establishing a new patroni restapi connection to postgres
```
После возобновления работы PostgreSQL Patroni обнаружил что не может обновить блокировку в DCS, перевел PostgreSQL в режим реплики.

### Прерывание работы Patroni

Отправляем Patroni сигнал SIGKILL (немедленно прекращает работу, невозможно игнорировать)

На ноде pg-db1 (на которой запущен мастер) выполняем команду:

```
sudo killall -KILL patroni
```
Мастер переключился на вторую ноду, Patroni сам не запустился. Postgres остался работать. Haproxy оборвал соединения со старым мастером.
После старта Patroni обнаружил что не имеет блокировки и при этом является мастером, выполнил demotion и rewind.

```
Feb 11 14:54:39 pg-db1 systemd: patroni.service: main process exited, code=killed, status=9/KILL
Feb 11 14:54:39 pg-db1 systemd: Unit patroni.service entered failed state.
Feb 11 14:54:39 pg-db1 systemd: patroni.service failed.
Feb 11 14:58:53 pg-db1 systemd: Started Runners to orchestrate a high-availability PostgreSQL.
Feb 11 14:58:54 pg-db1 patroni: 2024-02-11 14:58:54,171 INFO: Selected new etcd server http://pg-proxy:2379
Feb 11 14:58:54 pg-db1 patroni: 2024-02-11 14:58:54,180 INFO: No PostgreSQL configuration items changed, nothing to reload.
Feb 11 14:58:54 pg-db1 patroni: localhost:5432 - accepting connections
Feb 11 14:58:54 pg-db1 patroni: 2024-02-11 14:58:54,199 INFO: establishing a new patroni heartbeat connection to postgres
Feb 11 14:58:54 pg-db1 patroni: 2024-02-11 14:58:54,295 INFO: Changed synchronous_commit from off to False
Feb 11 14:58:54 pg-db1 patroni: 2024-02-11 14:58:54,309 INFO: Reloading PostgreSQL configuration.
Feb 11 14:58:54 pg-db1 patroni: server signaled
Feb 11 14:58:55 pg-db1 patroni: 2024-02-11 14:58:55,330 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 14:58:55 pg-db1 patroni: 2024-02-11 14:58:55,330 INFO: Demoting self (immediate-nolock)
Feb 11 14:58:55 pg-db1 patroni: 2024-02-11 14:58:55,400 INFO: demoting self because I do not have the lock and I was a leader
Feb 11 14:58:55 pg-db1 patroni: 2024-02-11 14:58:55,402 INFO: closed patroni connections to postgres
Feb 11 14:58:56 pg-db1 patroni: 2024-02-11 14:58:56,072 INFO: postmaster pid=2271
Feb 11 14:58:56 pg-db1 patroni: 2024-02-11 14:58:56.079 UTC [2271] LOG:  redirecting log output to logging collector process
Feb 11 14:58:56 pg-db1 patroni: 2024-02-11 14:58:56.079 UTC [2271] HINT:  Future log output will appear in directory "log".
Feb 11 14:58:56 pg-db1 patroni: localhost:5432 - rejecting connections
Feb 11 14:58:56 pg-db1 patroni: localhost:5432 - rejecting connections
Feb 11 14:58:57 pg-db1 patroni: localhost:5432 - accepting connections
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,162 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,163 INFO: establishing a new patroni heartbeat connection to postgres
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,189 INFO: Local timeline=20 lsn=0/A75A3EE0
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,210 INFO: primary_timeline=21
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,211 INFO: primary: history=17#0110/9FE3DF40#011no recovery target specified
Feb 11 14:58:57 pg-db1 patroni: 18#0110/A1000270#011no recovery target specified
Feb 11 14:58:57 pg-db1 patroni: 19#0110/A1000400#011no recovery target specified
Feb 11 14:58:57 pg-db1 patroni: 20#0110/A75A3DC0#011no recovery target specified
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,226 INFO: running pg_rewind from pg-db2
Feb 11 14:58:57 pg-db1 patroni: 2024-02-11 14:58:57,289 INFO: running pg_rewind from dbname=postgres user=rewind_user host=192.168.2.12 port=5432 target_session_attrs=read-write
Feb 11 14:59:07 pg-db1 patroni: 2024-02-11 14:59:07,161 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 14:59:07 pg-db1 patroni: 2024-02-11 14:59:07,163 INFO: running pg_rewind from pg-db2 in progress
Feb 11 14:59:17 pg-db1 patroni: 2024-02-11 14:59:17,161 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 14:59:17 pg-db1 patroni: 2024-02-11 14:59:17,163 INFO: running pg_rewind from pg-db2 in progress
Feb 11 14:59:19 pg-db1 patroni: 2024-02-11 14:59:19,696 INFO: pg_rewind exit code=0
Feb 11 14:59:19 pg-db1 patroni: 2024-02-11 14:59:19,696 INFO:  stdout=
Feb 11 14:59:19 pg-db1 patroni: 2024-02-11 14:59:19,697 INFO:  stderr=pg_rewind: servers diverged at WAL location 0/A75A3DC0 on timeline 20
Feb 11 14:59:19 pg-db1 patroni: pg_rewind: rewinding from last common checkpoint at 0/A1000468 on timeline 20
Feb 11 14:59:19 pg-db1 patroni: pg_rewind: Done!
```

### Приостановка работы Patroni

Отправляем Patroni сигнал SIGSTOP (немедленно приостанавливает работу не завершаясь, невозможно игнорировать)

На ноде pg-db1 (на которой запущен мастер) выполняем команду:
```
sudo killall -STOP patroni
```

Мастер переключился
```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Leader  | running   | 22 |           |
| pg-db2 | 192.168.2.12 | Replica | streaming | 22 |         0 |
+--------+--------------+---------+-----------+----+-----------+
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db2 | 192.168.2.12 | Replica | streaming | 22 |         0 |
+--------+--------------+---------+-----------+----+-----------+
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) ----+-----------+
| Member | Host         | Role   | State   | TL | Lag in MB |
+--------+--------------+--------+---------+----+-----------+
| pg-db2 | 192.168.2.12 | Leader | running | 23 |           |
+--------+--------------+--------+---------+----+-----------+
```

На pg-db2 Patroni обнаружил недоступность мастера, поднял реплику до мастера:
```
Feb 11 15:23:11 pg-db2 patroni: 2024-02-11 15:23:11,050 WARNING: Request failed to pg-db1: GET http://192.168.2.11:8008/patroni (HTTPConnectionPool(host='192.168.2.11', port=8008): Max retries exceeded with url: /patroni (Caused by ReadTimeoutError("HTTPConnectionPool(host='192.168.2.11', port=8008): Read timed out. (read timeout=2)",)))
Feb 11 15:23:11 pg-db2 patroni: 2024-02-11 15:23:11,054 WARNING: Could not activate Linux watchdog device: Can't open watchdog device: [Errno 13] Permission denied: '/dev/watchdog'
Feb 11 15:23:11 pg-db2 patroni: server promoting
Feb 11 15:23:11 pg-db2 patroni: 2024-02-11 15:23:11,091 INFO: promoted self to leader by acquiring session lock
Feb 11 15:23:12 pg-db2 patroni: 2024-02-11 15:23:12,123 INFO: no action. I am (pg-db2), the leader with the lock
```

Возобновляем работу Patroni:
```
sudo killall -CONT patroni
```
Видим, что на pg-db1 кластер запустился в режиме реплики.

[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) --+----+-----------+
| Member | Host         | Role    | State     | TL | Lag in MB |
+--------+--------------+---------+-----------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | streaming | 24 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running   | 24 |           |
+--------+--------------+---------+-----------+----+-----------+

В логах на pg-db1:
```
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,792 WARNING: Exception happened during processing of request from 192.168.2.10:47226
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,794 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,794 INFO: Demoting self (immediate-nolock)
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,797 WARNING: Exception happened during processing of request from 192.168.2.10:47240
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,906 INFO: demoting self because I do not have the lock and I was a leader
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,907 INFO: closed patroni connections to postgres
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,908 WARNING: Loop time exceeded, rescheduling immediately.
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,910 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 15:25:29 pg-db1 patroni: 2024-02-11 15:25:29,915 INFO: starting after demotion in progress

Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,631 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,632 INFO: establishing a new patroni heartbeat connection to postgres
Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,656 INFO: Local timeline=22 lsn=0/A75ADEE8
Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,670 INFO: establishing a new patroni restapi connection to postgres
Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,682 INFO: primary_timeline=23
Feb 11 15:25:31 pg-db1 patroni: 2024-02-11 15:25:31,682 INFO: primary: history=19#0110/A1000400#011no recovery target specified
Feb 11 15:25:31 pg-db1 patroni: 20#0110/A75A3DC0#011no recovery target specified
Feb 11 15:25:31 pg-db1 patroni: 21#0110/A75A9A88#011no recovery target specified
Feb 11 15:25:31 pg-db1 patroni: 22#0110/A75ADEE8#011no recovery target specified
```

### Остановка etcd

```
sudo systemctl stop etcd
```

Пытаемся посмотреть состояние кластера, получаем сообщение об ошибке:
```
patronictl -c /etc/patroni/patroni.yml list
2024-02-11 15:51:52,857 - WARNING - Retrying (Retry(total=1, connect=None, read=None, redirect=0, status=None)) after connection broken by 'NewConnectionError('<urllib3.connection.HTTPConnection object at 0x7f1418635630>: Failed to establish a new connection: [Errno 111] Connection refused',)': /v2/machines
```

Оба сервера теперь в режиме реплики:
```
http http://pg-db1:8008/
HTTP/1.0 503 Service Unavailable
Content-Type: application/json
Date: Sun, 11 Feb 2024 15:59:52 GMT
Server: BaseHTTP/0.6 Python/3.6.8

{
    "cluster_unlocked": true,
    "database_system_identifier": "7331748476172603812",
    "dcs_last_seen": 1707666686,
    "patroni": {
        "name": "pg-db1",
        "scope": "pg_cluster",
        "version": "3.2.2"
    },
    "postmaster_start_time": "2024-02-11 15:51:52.548061+00:00",
    "replication": [
        {
            "application_name": "pg-db2",
            "client_addr": "192.168.2.12",
            "state": "streaming",
            "sync_priority": 0,
            "sync_state": "async",
            "usename": "replicator"
        }
    ],
    "role": "replica",
    "server_version": 150005,
    "state": "running",
    "timeline": 24,
    "xlog": {
        "paused": false,
        "received_location": 3584279888,
        "replayed_location": 3584279888,
        "replayed_timestamp": null
    }
}


http http://pg-db2:8008/
HTTP/1.0 503 Service Unavailable
Content-Type: application/json
Date: Sun, 11 Feb 2024 16:00:26 GMT
Server: BaseHTTP/0.6 Python/3.6.8

{
    "cluster_unlocked": true,
    "database_system_identifier": "7331748476172603812",
    "dcs_last_seen": 1707666686,
    "patroni": {
        "name": "pg-db2",
        "scope": "pg_cluster",
        "version": "3.2.2"
    },
    "postmaster_start_time": "2024-02-11 15:34:53.879199+00:00",
    "replication_state": "streaming",
    "role": "replica",
    "server_version": 150005,
    "state": "running",
    "timeline": 24,
    "xlog": {
        "paused": false,
        "received_location": 3584279888,
        "replayed_location": 3584279888,
        "replayed_timestamp": "2024-02-11 15:48:46.155610+00:00"
    }
}

```

Записи в логах:
```
Feb 11 15:51:44 pg-db1 patroni: 2024-02-11 15:51:43,888 ERROR: get_cluster
Feb 11 15:51:44 pg-db1 patroni: Traceback (most recent call last):
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 738, in _load_cluster
Feb 11 15:51:44 pg-db1 patroni: cluster = loader(path)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 714, in _cluster_loader
Feb 11 15:51:44 pg-db1 patroni: result = self.retry(self._client.read, path, recursive=True, quorum=self._ctl)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 495, in retry
Feb 11 15:51:44 pg-db1 patroni: return retry(method, *args, **kwargs)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/utils.py", line 613, in __call__
Feb 11 15:51:44 pg-db1 patroni: return func(*args, **kwargs)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/etcd/client.py", line 597, in read
Feb 11 15:51:44 pg-db1 patroni: timeout=timeout)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 301, in api_execute
Feb 11 15:51:44 pg-db1 patroni: raise ex
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 282, in api_execute
Feb 11 15:51:44 pg-db1 patroni: response = self._do_http_request(retry, machines_cache, request_executor, method, path, **kwargs)
Feb 11 15:51:44 pg-db1 patroni: File "/usr/lib/python3.6/site-packages/patroni/dcs/etcd.py", line 257, in _do_http_request
Feb 11 15:51:44 pg-db1 patroni: raise etcd.EtcdConnectionFailed('No more machines in the cluster')
Feb 11 15:51:44 pg-db1 patroni: etcd.EtcdConnectionFailed: No more machines in the cluster
Feb 11 15:51:44 pg-db1 patroni: 2024-02-11 15:51:44,045 ERROR: Error communicating with DCS
Feb 11 15:51:44 pg-db1 patroni: 2024-02-11 15:51:44,046 INFO: demoting self because DCS is not accessible and I was a leader
Feb 11 15:51:44 pg-db1 patroni: 2024-02-11 15:51:44,046 INFO: Demoting self (offline)
```
На мастере Patroni обнаружил недоступность etcd и перевёл  PostgreSQL в режим реплики. На реплике PostgreSQL также остался в режиме реплики. На обоих серверах Patroni пытается подключиться к etcd.

Запускаем etcd
```
sudo systemctl start etcd
```

Проверяем состояние кластера:

```
[yc-user@pg-db1 ~]$ patronictl -c /etc/patroni/patroni.yml list
+ Cluster: pg_cluster (7331748476172603812) +----+-----------+
| Member | Host         | Role    | State   | TL | Lag in MB |
+--------+--------------+---------+---------+----+-----------+
| pg-db1 | 192.168.2.11 | Replica | running | 25 |         0 |
| pg-db2 | 192.168.2.12 | Leader  | running | 25 |           |
+--------+--------------+---------+---------+----+-----------+
```

Записи в логах:
```
Feb 11 16:05:06 pg-db2 patroni: 2024-02-11 16:05:06,602 ERROR: Could not get the list of servers, maybe you provided the wrong host(s) to connect to?
Feb 11 16:05:08 pg-db2 patroni: 2024-02-11 16:05:08,396 INFO: Got response from pg-db1 http://192.168.2.11:8008/patroni: {"state": "running", "postmaster_start_time": "2024-02-11 15:51:52.54
8061+00:00", "role": "replica", "server_version": 150005, "xlog": {"received_location": 3584279888, "replayed_location": 3584279888, "replayed_timestamp": null, "paused": false}, "timeline":
 24, "replication": [{"usename": "replicator", "application_name": "pg-db2", "client_addr": "192.168.2.12", "state": "streaming", "sync_state": "async", "sync_priority": 0}], "cluster_unlock
ed": true, "dcs_last_seen": 1707666686, "database_system_identifier": "7331748476172603812", "patroni": {"version": "3.2.2", "scope": "pg_cluster", "name": "pg-db1"}}
Feb 11 16:05:08 pg-db2 patroni: 2024-02-11 16:05:08,489 INFO: promoted self to leader by acquiring session lock
Feb 11 16:05:08 pg-db2 patroni: server promoting


Feb 11 16:05:06 pg-db1 patroni: 2024-02-11 16:05:06,578 ERROR: Could not get the list of servers, maybe you provided the wrong host(s) to connect to?
Feb 11 16:05:06 pg-db1 patroni: 2024-02-11 16:05:06,578 ERROR: Error communicating with DCS
Feb 11 16:05:06 pg-db1 patroni: 2024-02-11 16:05:06,579 INFO: DCS is not accessible
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,591 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,600 INFO: Local timeline=24 lsn=0/D5A3C550
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,622 INFO: primary_timeline=25
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,622 INFO: primary: history=21#0110/A75A9A88#011no recovery target specified
Feb 11 16:05:16 pg-db1 patroni: 22#0110/A75ADEE8#011no recovery target specified
Feb 11 16:05:16 pg-db1 patroni: 23#0110/A75AE0C8#011no recovery target specified
Feb 11 16:05:16 pg-db1 patroni: 24#0110/D5A3C550#011no recovery target specified
Feb 11 16:05:16 pg-db1 patroni: server signaled
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,663 INFO: Dropped unknown replication slot 'pg_db2'
Feb 11 16:05:16 pg-db1 patroni: 2024-02-11 16:05:16,684 INFO: no action. I am (pg-db1), a secondary, and following a leader (pg-db2)
Feb 11 16:05:19 pg-db1 patroni: 2024-02-11 16:05:19,520 INFO: Lock owner: pg-db2; I am pg-db1
Feb 11 16:05:19 pg-db1 patroni: 2024-02-11 16:05:19,529 INFO: Local timeline=24 lsn=0/D5A3C550
Feb 11 16:05:19 pg-db1 patroni: 2024-02-11 16:05:19,551 INFO: primary_timeline=25
Feb 11 16:05:19 pg-db1 patroni: 2024-02-11 16:05:19,551 INFO: primary: history=21#0110/A75A9A88#011no recovery target specified
Feb 11 16:05:19 pg-db1 patroni: 22#0110/A75ADEE8#011no recovery target specified
Feb 11 16:05:19 pg-db1 patroni: 23#0110/A75AE0C8#011no recovery target specified
Feb 11 16:05:19 pg-db1 patroni: 24#0110/D5A3C550#011no recovery target specified
Feb 11 16:05:19 pg-db1 patroni: 2024-02-11 16:05:19,566 INFO: no action. I am (pg-db1), a secondary, and following a leader (pg-db2)
```

Patroni обнаружил что etcd теперь доступен, выбрал нового мастера. Кластер работает в обычном режиме.

**При недоступности etcd все сервера перешли в режим реплики. После восстановления доступности etcd был выбран новый мастер и кластер продолжил работу в обычном режиме. Splitbrain-а и повреждения данных не произошло, работа кластера была восстановлена автоматически, перезапускать PostgreSQL и Patroni не потребовалось**

## Тестирование работы кластера под нагрузкой

Подать нагрузку с помощью tpcc-sysbench
```
./tpcc.lua run --pgsql-host=192.168.2.10 --pgsql-port=5000 --pgsql-user=sbtest --pgsql-db=sbtest --pgsql-password=otus135 --time=120 --report-interval=1 --tables=1 --scale=10 --use_fk=0 --db-driver=pgsql
```

Результаты теста:
```
SQL statistics:
    queries performed:
        read:                            157612
        write:                           163409
        other:                           24606
        total:                           345627
    transactions:                        12302  (102.51 per sec.)
    queries:                             345627 (2880.10 per sec.)
    ignored errors:                      64     (0.53 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0037s
    total number of events:              12302

Latency (ms):
         min:                                    1.38
         avg:                                    9.75
         max:                                  237.68
         95th percentile:                       23.52
         sum:                               119957.21

Threads fairness:
    events (avg/stddev):           12302.0000/0.00
    execution time (avg/stddev):   119.9572/0.00
```

Проблем, падений сервисов, отставания репликации во время теста не наблюдалось.