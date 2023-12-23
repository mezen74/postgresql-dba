# Домашнее задание #8

## Настройка PostgreSQL


- развернуть виртуальную машину любым удобным способом
- поставить на неё PostgreSQL 15 любым способом
- настроить кластер PostgreSQL 15 на максимальную производительность не обращая внимание на возможные проблемы с надежностью в случае аварийной перезагрузки виртуальной машины
- нагрузить кластер через утилиту через утилиту pgbench (https://postgrespro.ru/docs/postgrespro/14/pgbench)
- написать какого значения tps удалось достичь, показать какие параметры в какие значения устанавливали и почему

__Тесты выполняла на ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB__

__Результат тестирования с настройками по умолчанию:__
```
number of transactions actually processed: 7768
number of failed transactions: 0 (0.000%)
latency average = 386.993 ms
latency stddev = 307.148 ms
initial connection time = 82.037 ms
tps = 128.772926 (without initial connection time)
```

__Результат тестирования после изменения настроек:__
```
number of transactions actually processed: 135296
number of failed transactions: 0 (0.000%)
latency average = 22.165 ms
latency stddev = 11.854 ms
initial connection time = 76.732 ms
tps = 2253.865152 (without initial connection time)
```

__Таким образом, после настойки параметров удалось достичь значения 2253 tps. С параметрами по умолчанию было 128 tps.__

Меняла следующие параметры:

`shared_buffers = 1GB` - данный параметр рекомендуется делать от 25% до 40% объема памяти. Пробовала выставлять разные значения от 25% до 40% - заметной разницы в результатах теста не было. Оставила 25%

`effective_cache_size` = 3GB - Этот параметр не влияет на размер разделяемой памяти, выделяемой PostgreSQL, и не задаёт размер резервируемого в ядре дискового кеша; он используется только в качестве ориентировочной оценки. Влияет на оценку стоимости использования планировщиком индекса; чем выше это значение, тем больше вероятность, что будет применяться сканирование по индексу, чем ниже, тем более вероятно, что будет выбрано последовательное сканирование. 

`work_mem = 8MB` - Рекомендуется 1/32 от общего объема памяти, но нужно следить чтобы work_mem * max_connections < RAM. Слишком большое значение может привести к тому, что будет срабатывать OOM Killer. Этот параметр возможно задавать для конкретной сессии. В тестах видимой разницы при разных значениях не заметила.

`maintenance_work_mem = 256MB` - рекомендация 1/16 объема оперативной памяти. Увеличение может привести к ускорению операций очистки БД. В тестах видимой разницы при разных значениях не заметила.

`random_page_cost = 1.1` - рекомендуемое значение для SSD-дисков

`effective_io_concurrency = 200` - рекомендуемое значение. В тестах видимой разницы при разных значениях не заметила.

`checkpoint_completion_target = 0.9` - рекомендуемое значение, чтобы сгладить нагрузку на диск при выполнении контрольных точек.

`wal_buffers = 16MB` - увеличение поможет сгладить скачки во времени ответа после выполнения каждой контрольной точки, но может привести к увеличению времени восстановления после сбоя.

`min_wal_size = 4GB`, `max_wal_size = 16GB` - Чтобы уменьшить накладные расходы на выполнение контрольных точек

`synchronous_commit = off`  - Отключение значительно уменьшает время отклика и повышает производительность системы, потому что фиксация изменений не ждет физической записи на диск. Но при этом возрастает риск потери изменений в случае сбоя.


### Задание со * 

аналогично протестировать через утилиту https://github.com/Percona-Lab/sysbench-tpcc (требует установки https://github.com/akopytov/sysbench)

Установка sysbench
```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.deb.sh | sudo bash
sudo apt -y install sysbench
```
Создание пользователя для тестов:
```
CREATE USER sbtest WITH PASSWORD 'password';
CREATE DATABASE sbtest;
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
```

Установка sysbench-tpcc
```
git clone https://github.com/Percona-Lab/sysbench-tpcc
```

Подготовка таблиц
```
./tpcc.lua --pgsql-user=sbtest --pgsql-db=sbtest --time=120 --threads=56 --report-interval=1 --tables=1 --scale=10 --use_fk=0  --trx_level=RC --db-driver=pgsql prepare
```

__Запуск теста__
```
./tpcc.lua --pgsql-user=sbtest --pgsql-db=sbtest --pgsql-password=password --time=120 --report-interval=1 --tables=1 --scale=10 --use_fk=0  --trx_level=RC  --db-driver=pgsql run
```

__Результат теста с параметрами по умолчанию:__
```
SQL statistics:
    queries performed:
        read:                            105553
        write:                           109768
        other:                           16380
        total:                           231701
    transactions:                        8189   (68.23 per sec.)
    queries:                             231701 (1930.51 per sec.)
    ignored errors:                      35     (0.29 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0187s
    total number of events:              8189

Latency (ms):
         min:                                    0.71
         avg:                                   14.65
         max:                                  278.88
         95th percentile:                       28.67
         sum:                               119979.89

Threads fairness:
    events (avg/stddev):           8189.0000/0.00
    execution time (avg/stddev):   119.9799/0.00
```


__Результат тестирования после изменения настроек:__

```
SQL statistics:
    queries performed:
        read:                            396050
        write:                           411576
        other:                           60954
        total:                           868580
    transactions:                        30476  (253.95 per sec.)
    queries:                             868580 (7237.63 per sec.)
    ignored errors:                      122    (1.02 per sec.)
    reconnects:                          0      (0.00 per sec.)

General statistics:
    total time:                          120.0074s
    total number of events:              30476

Latency (ms):
         min:                                    0.51
         avg:                                    3.94
         max:                                   24.86
         95th percentile:                       10.27
         sum:                               119946.94

Threads fairness:
    events (avg/stddev):           30476.0000/0.00
    execution time (avg/stddev):   119.9469/0.00

```

Таким образом, после настойки параметров удалось достичь значения 253.95 tps и 7237.63 qps. С параметрами по умолчанию было 68.23 tps и 1930.51 qps.
