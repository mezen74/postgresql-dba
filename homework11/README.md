# Домашнее задание #11

## Работа с индексами

1. Создать индекс к какой-либо из таблиц вашей БД. Прислать текстом результат команды explain, в которой используется данный индекс.

__Создала индекс по полю postal_code. Этот индекс позволяет оптимизировать поиск адресов по почтовому индексу__
```
create index on address(postal_code);
```

__Результат explain без использования индекса:__
```
dvdrental=# explain select address from address where postal_code = '90920';
                       QUERY PLAN                        
---------------------------------------------------------
 Seq Scan on address  (cost=0.00..15.54 rows=1 width=20)
   Filter: ((postal_code)::text = '90920'::text)
```

__Результат explain с использованием индекса:__
```
dvdrental=# explain select address from address where postal_code = '90920';
                                       QUERY PLAN                                       
-------------------------------------------------------------------------
 Index Scan using address_postal_code_idx on address  (cost=0.28..2.49 rows=1 width=20)
   Index Cond: ((postal_code)::text = '90920'::text)
(2 строки)
```


2. Реализовать индекс для полнотекстового поиска

__Создала индекс типа GIN по полю 'description'. Он позволяет оптимизировать поиск по ключевым словам в описании фильма.
При создании индекса использовала функцию to_tsvector с двумя аргументами. В выражениях, определяющих индексы, можно использовать только функции, в которых явно задаётся имя конфигурации текстового поиска. В данном случае имя конфигураци - english. Так как при создании индекса использовалась версия to_tsvector с двумя аргументами, то он будет использоваться только в запросах, где также to_tsvector используется с двумя аргументами и передаётся имя той же конфигурации.__

```
dvdrental=# create index film_descr_idx on film using gin (to_tsvector('english', description));
```
__Результат explain без использования индекса:__
```
dvdrental=# explain select title, description from film where to_tsvector('english', description) @@ to_tsquery('cat & dog');
                                         QUERY PLAN                                          
-------------------------------------------------------------------------
 Seq Scan on film  (cost=0.00..566.50 rows=1 width=109)
   Filter: (to_tsvector('english'::regconfig, description) @@ to_tsquery('cat & dog'::text))
(2 строки)
```

__Результат explain с использованием индекса:__
```
dvdrental=# explain select title, description from film where to_tsvector('english', description) @@ to_tsquery('cat & dog');
                                              QUERY PLAN                                               
-------------------------------------------------------------------------
 Bitmap Heap Scan on film  (cost=4.51..6.12 rows=1 width=109)
   Recheck Cond: (to_tsvector('english'::regconfig, description) @@ to_tsquery('cat & dog'::text))
   ->  Bitmap Index Scan on film_descr_idx  (cost=0.00..4.51 rows=1 width=0)
         Index Cond: (to_tsvector('english'::regconfig, description) @@ to_tsquery('cat & dog'::text))
(4 строки)
```

3. Реализовать индекс на часть таблицы или индекс
на поле с функцией

__Создала индекс на поле с функцией для таблицы actor. Данный индекс позволяет оптимизировать поиск по фамилии актёра без учёта регистра.__
```
dvdrental=# create index idx_last_name_lower on actor (lower(last_name));
CREATE INDEX

dvdrental=# explain select first_name, last_name from actor where lower(last_name) = 'guiness';
                                    QUERY PLAN                                    
-------------------------------------------------------------------------
 Index Scan using idx_last_name_lower on actor  (cost=0.14..2.36 rows=1 width=13)
   Index Cond: (lower((last_name)::text) = 'guiness'::text)
(2 строки)
```

__Создала индекс на часть таблицы payment. Этот индекс позволяет быстро выполнить поиск платежей с нулевой суммой. Записей о таких платежах в таблице относительно немного, поэтому использование индекса в данном случае эффективно. Он будет использоваться во всех запросах, в которых выбираются платежи с нулевой суммой, независимо от других условий фильтрации.__
```
dvdrental=# create index idx_payment_amount on payment (payment_id) where amount = 0;
CREATE INDEX

dvdrental=# explain select * from payment where staff_id = 2 and amount = 0;
                                     QUERY PLAN                                     
-------------------------------------------------------------------------
 Index Scan using idx_payment_amount on payment  (cost=0.14..3.86 rows=12 width=26)
   Filter: (staff_id = 2)
(2 строки)
```

4. Создать индекс на несколько полей

__Создала индекс для таблицы film по двум полям. Он позволяет оптимизировать поиск по комбинации рейтинга фильма и времени аренды фильма.__

```
dvdrental=# create index idx_film_rating_rental on film(rating, rental_duration);
CREATE INDEX

dvdrental=# explain select title from film where rating = 'PG-13' and rental_duration = 7;
                                      QUERY PLAN                                      
-------------------------------------------------------------------------

 Bitmap Heap Scan on film  (cost=1.69..34.09 rows=43 width=15)
   Recheck Cond: ((rating = 'PG-13'::mpaa_rating) AND (rental_duration = 7))
   ->  Bitmap Index Scan on idx_film_rating_rental  (cost=0.00..1.68 rows=43 width=0)
         Index Cond: ((rating = 'PG-13'::mpaa_rating) AND (rental_duration = 7))
(4 строки)
```

__Порядок указания полей при создании индекса имеет значение. В данном случае индекс будет использоваться при поиске фильма по рейтингу и времени аренды, а также только по рейтингу. При поиске только по времени аренды индекс использоваться не будет:__

```
dvdrental=# explain select title from film where rating = 'PG-13';
                                      QUERY PLAN                                       
-------------------------------------------------------------------------
 Bitmap Heap Scan on film  (cost=2.98..59.77 rows=223 width=15)
   Recheck Cond: (rating = 'PG-13'::mpaa_rating)
   ->  Bitmap Index Scan on idx_film_rating_rental  (cost=0.00..2.92 rows=223 width=0)
         Index Cond: (rating = 'PG-13'::mpaa_rating)
(4 строки)

dvdrental=# explain select title from film where rental_duration = 7;
                       QUERY PLAN                       
--------------------------------------------------------
 Seq Scan on film  (cost=0.00..66.50 rows=191 width=15)
   Filter: (rental_duration = 7)
(2 строки)
```