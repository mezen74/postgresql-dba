# Домашнее задание #5

## MVCC, vacuum и autovacuum


- Создать инстанс ВМ с 2 ядрами и 4 Гб ОЗУ и SSD 10GB
- Установить на него PostgreSQL 15 с дефолтными настройками
- Создать БД для тестов: выполнить pgbench -i postgres
- Запустить pgbench -c8 -P 6 -T 60 -U postgres postgres
- Применить параметры настройки PostgreSQL из прикрепленного к материалам занятия файла
- Протестировать заново
- Что изменилось и почему?

**Выполняла тестирование в Яндекс-Облаке. Мне не удалось получить сколько-нибудь существенных различий в результатах. Повторяла тесты по нескольку раз, хотелось получить воспроизводимые результаты, но иногда значения tps были выше с настройками из файла, а иногда - с настройками по умолчанию. Пробовала увеличивать время тестирования до 10 минут - тоже явных различий в результатах не получила.
После изменения параметров перезапускала Postgres, подключалась с помощью psql и проверяла текущие значения параметров, поэтому ситуация что изменения параметров не применялись - исключена.**

**Также настораживает фраза из документации PostgreSQL, из раздела Good Practices: "If autovacuum is enabled it can result in unpredictable changes in measured performance. - Если же включена автоочистка, это может быть чревато непредсказуемыми изменениями оценок производительности." Попробовала отключить autovacuum и повторить тесты - изменений не увидела.**


- Создать таблицу с текстовым полем и заполнить случайными или сгенерированными данными в размере 1млн строк
```
create table test (test_id serial, test_text char(100));
insert into test (test_text) SELECT md5(random()::text) FROM generate_series(1,1000000);
```
- Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 135 MB
```
- 5 раз обновить все строчки и добавить к каждой строчке любой символ
- Посмотреть количество мертвых строчек в таблице и когда последний раз приходил автовакуум
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |    1000000 |    5000000 |    499 | 2023-10-29 19:18:06.418734+00
(1 строка)

```
- Подождать некоторое время, проверяя, пришел ли автовакуум
```
postgres=# SELECT relname, n_live_tup, n_dead_tup, trunc(100*n_dead_tup/(n_live_tup+1))::float "ratio%", last_autovacuum FROM pg_stat_user_TABLEs WHERE relname = 'test';
 relname | n_live_tup | n_dead_tup | ratio% |        last_autovacuum        
---------+------------+------------+--------+-------------------------------
 test    |     990007 |          0 |      0 | 2023-10-29 19:21:49.604389+00
(1 строка)

postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 808 MB
```

- 5 раз обновить все строчки и добавить к каждой строчке любой символ
- Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 808 MB

```
- Отключить Автовакуум на конкретной таблице
```
alter table test set (autovacuum_enabled = off);
```
- 10 раз обновить все строчки и добавить к каждой строчке любой символ

- Посмотреть размер файла с таблицей
```
postgres=# SELECT pg_size_pretty(pg_total_relation_size('test'));
 pg_size_pretty 
----------------
 1482 MB
```

- Объясните полученный результат

**Очистка не уменьшает размер файлов с таблицами, место не возвращается операционной системе. Но освободившееся в ходе очистки пространство в страницах используется для размещения новых данных. Поэтому файл данных при первом обновлении строчек вырос, а при втором, которое было после того как отработал autovacuum, размер файла не изменился.**

**При отключенном autovacuum файл с таблицей вырос после обновления строк, так как очистка не работала, место в страницах, занятое старыми версиями строк, не освобождалось, и оно не могло быть использовано для новых данных.**

- Не забудьте включить автовакуум)
```
 alter table test set (autovacuum_enabled = on);
```
    
## Задание со *:
Написать анонимную процедуру, в которой в цикле 10 раз обновятся все строчки в искомой таблице.
Не забыть вывести номер шага цикла.
```
DO $$
BEGIN
  FOR i IN 1..10
  LOOP
    RAISE NOTICE 'Step = %', i;
	update test set test_text = test_text || i::text;
  END LOOP;
END;
$$
;
```

