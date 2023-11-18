# Домашнее задание #7

## Механизм блокировок


1. Настройте сервер так, чтобы в журнал сообщений сбрасывалась информация о блокировках, удерживаемых более 200 миллисекунд. Воспроизведите ситуацию, при которой в журнале появятся такие сообщения.
```
-- Включаем запись в журнал событий, когда ожидание получения блокировки больше, чем указано в deadlock_timeout
ALTER SYSTEM SET log_lock_waits = on;

-- Меняем значение deadlock_timeout (по умолчанию 1 секунда) на 200 миллисекунд, как требуется по условию задачи.
ALTER SYSTEM SET deadlock_timeout = 200;

-- Применяем изменения 
select pg_reload_conf();

-- Сессия #1
begin;
update accounts set amount = amount + 10 where acc_no = 1;
-- commit пока не выполняем

-- Сессия #2
begin;
update accounts set amount = amount + 100 where acc_no = 1;
-- транзакция ожидает блокировку, установленную транзакцией в первой сессии

-- Сессия #1
commit;
-- Сессия #2
commit;

-- Записи из лога:
2023-11-15 14:46:45.279 UTC [1304] postgres@homework07 СООБЩЕНИЕ:  процесс 1304 продолжает ожидать в режиме ShareLock блокировку "транзакция 738" в течение 200.184 мс
2023-11-15 14:46:45.279 UTC [1304] postgres@homework07 ПОДРОБНОСТИ:  Process holding the lock: 1387. Wait queue: 1304.
2023-11-15 14:46:45.279 UTC [1304] postgres@homework07 КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "accounts"
2023-11-15 14:46:45.279 UTC [1304] postgres@homework07 ОПЕРАТОР:  update accounts set amount = amount + 100 where acc_no = 1;
2023-11-15 14:46:59.479 UTC [1304] postgres@homework07 СООБЩЕНИЕ:  процесс 1304 получил в режиме ShareLock блокировку "транзакция 738" через 14400.995 мс
2023-11-15 14:46:59.479 UTC [1304] postgres@homework07 КОНТЕКСТ:  при изменении кортежа (0,1) в отношении "accounts"
2023-11-15 14:46:59.479 UTC [1304] postgres@homework07 ОПЕРАТОР:  update accounts set amount = amount + 100 where acc_no = 1;
```

2. Смоделируйте ситуацию обновления одной и той же строки тремя командами UPDATE в разных сеансах. Изучите возникшие блокировки в представлении pg_locks и убедитесь, что все они понятны. Пришлите список блокировок и объясните, что значит каждая.
```
-- Возникли такие блокировки:

-- Сессия #1
 pid  |   locktype    |  lockid  |       mode       | granted 
------+---------------+----------+------------------+---------
 1304 | relation      | accounts | RowExclusiveLock | t
 1304 | transactionid | 746      | ExclusiveLock    | t


-- Сессия #2
 pid  |   locktype    |   lockid   |       mode       | granted 
------+---------------+------------+------------------+---------
 2406 | relation      | accounts   | RowExclusiveLock | t
 2406 | tuple         | accounts:3 | ExclusiveLock    | t
 2406 | transactionid | 746        | ShareLock        | f
 2406 | transactionid | 748        | ExclusiveLock    | t


-- Сессия #3
 pid  |   locktype    |   lockid   |       mode       | granted 
------+---------------+------------+------------------+---------
 2528 | relation      | accounts   | RowExclusiveLock | t
 2528 | tuple         | accounts:3 | ExclusiveLock    | f
 2528 | transactionid | 749        | ExclusiveLock    | t
```
### Пояснения по возникшим блокировкам:
Все три транзакции удерживают по две блокировки: 
- Блокировку собственного номера transactionid в режиме ExclusiveLock - такую блокировку удерживает каждая транзакция;
- Блокировку изменяемой таблицы accounts в режиме RowExclusiveLock. Блокировку в этом режиме получает любая команда, которая изменяет данные в таблице. Этот режим не конфликтует сам с собой, поэтому блокировку получили все три транзакции.

Вторая транзакция удерживает эксклюзивную блокировку (ExclusiveLock) версии строки (tuple), в которую она вносит изменения, а также запрашивает блокировку в режиме ShareLock на номер первой транзакции, тем самым ожидая завершения первой транзакции, которая блокирует нужную ей строку.

Третья транзакция пытается установить блокировку версии строки (tuple) в режиме ExclusiveLock, но данная версия строки уже заблокирована в режиме ExclusiveLock второй транзакцией, поэтому третья транзакция ожидает снятия блокировки версии строки второй транзакцией.


3. Воспроизведите взаимоблокировку трех транзакций. Можно ли разобраться в ситуации постфактум, изучая журнал сообщений?

```
-- Первая транзакция выполняет перевод со счёта 1 на счёт 2 суммы 100 рублей

-- Сессия #1:
-- Транзакция уменьшает сумму счёта 1 на 100 рублей
UPDATE accounts SET amount = amount - 100.00 WHERE acc_no = 1;

-- В это время вторая транзакция выполняет перевод со счёта 2 на счёт 3 суммы 150 рублей

-- Сессия #2:
-- Транзакция уменьшает сумму счета 2 на 150 рублей
UPDATE accounts SET amount = amount - 150.00 WHERE acc_no = 2;

-- Третья транзакция выполняет перевод со счета 3 на счет 1 суммы 200 рублей

-- Сессия #3:
-- Транзакция уменьшает сумму счета 3 на 200 рублей
UPDATE accounts SET amount = amount - 200.00 WHERE acc_no = 3;

-- Сессия #1:
-- Транзакция увеличивает сумму счета 2 на 100 рублей
UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
-- "Зависает" в ожидании завершения транзакции из сессии #2

-- Сессия #2: 
-- Транзакция увеличивает сумму счета 3 на 150 рублей
UPDATE accounts SET amount = amount + 150.00 WHERE acc_no = 3;
-- "Зависает" в ожидании завершения транзакции из сессии #3

-- Сессия #3: 
-- Транзакция увеличивает сумму счета 1 на 200 рублей
UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
-- "Зависает" в ожидании завершения транзакции из сессии #1
-- Проходит время deadlock_timeout, обнаруживается взаимоблокировка, транзакция принудительно прерывается, освобождаются захваченные ей блокировки, две другие транзакции могут продолжить работу.

```

Разобраться постфактум по записям в журнале возможно. 
В журнале остались такие записи:
```
2023-11-18 14:42:21.358 UTC [1975] postgres@homework07 ОШИБКА:  обнаружена взаимоблокировка
2023-11-18 14:42:21.358 UTC [1975] postgres@homework07 ПОДРОБНОСТИ:  Процесс 1975 ожидает в режиме ShareLock блокировку "транзакция 757"; заблокирован процессом 1564.
	Процесс 1564 ожидает в режиме ShareLock блокировку "транзакция 758"; заблокирован процессом 1852.
	Процесс 1852 ожидает в режиме ShareLock блокировку "транзакция 759"; заблокирован процессом 1975.
	Процесс 1975: UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;
	Процесс 1564: UPDATE accounts SET amount = amount + 100.00 WHERE acc_no = 2;
	Процесс 1852: UPDATE accounts SET amount = amount + 150.00 WHERE acc_no = 3;
2023-11-18 14:42:21.358 UTC [1975] postgres@homework07 ПОДСКАЗКА:  Подробности запроса смотрите в протоколе сервера.
2023-11-18 14:42:21.358 UTC [1975] postgres@homework07 КОНТЕКСТ:  при изменении кортежа (0,17) в отношении "accounts"
2023-11-18 14:42:21.358 UTC [1975] postgres@homework07 ОПЕРАТОР:  UPDATE accounts SET amount = amount + 200.00 WHERE acc_no = 1;

```
**По записям видно, какие запросы выполнялись в транзакциях в момент возникновения блокировки, какие типы блокировок транзакции пытались захватить, кто кого блокировал. Также видно, в какой сессии транзакция была прервана и какой запрос в это время в ней выполнялся.**

4. Могут ли две транзакции, выполняющие единственную команду UPDATE одной и той же таблицы (без where), заблокировать друг друга?

**Да, такая ситуация возможна. Команда UPDATE блокирует строки по мере их обновления (а не все сразу), и это происходит не одномоментно. Поэтому если одна команда UPDATE обновляет несколько строк в одном порядке, а другая в другом, то они могут заблокировать друг друга. Такие ситуации реально возникают в нагруженных системах.**

## Задание со звездочкой*

Попробуйте воспроизвести такую ситуацию. 

Пример, как воспроизвести такую ситуацию, приведён в книге: Рогов Е.В.
PostgreSQL 15 изнутри. — М.: ДМК Пресс, 2023. — 662 с. (https://postgrespro.ru/education/books/internals), страница 275

Суть примера в том, что в одном сеансе таблица обновляется с планом выполнения Seq Scan, а в другом сеансе - с планом сканирование по индексу. При этом таблица подготовлена так, что записи на табличной странице расположены в порядке возрастания суммы, а индекс построен по полю суммы в порядке убывания. Соответственно в одном сеансе обновление записей выполняется в порядке возрастания, а в другом - в порядке убывания суммы. Кроме того, используется "замедляющая" функция, которая выполняет обновление с задержкой в 1 секунду.
Каждая из транзакций успевает обновить и заблокировать часть записей, а потом наступает взаимоблокировка.

Воспроизвести пример из книги у меня получилось с нескольких попыток: я увеличила количество строк в таблице с трёх до десяти, а также увеличила время задержки в замедляющей функции. Тогда получилось воспроизвести взаимоблокировку. По-видимому, с тремя записями в таблице и временем задержки в 1 секунду первая транзакция успевала отработать раньше, чем я во втором сеансе запускала вторую.

Пример выглядит искусственным, но реально в работающих системах могут быть ситуации, когда транзакции выполняют обновление записей в разном порядке, а при большом количестве записей время обновления может быть довольно продолжительным, и такая ситуация возможна.