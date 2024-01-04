# Домашнее задание #11

## Работа с индексами

1. Создать индекс к какой-либо из таблиц вашей БД

__Результат explain без использования индекса:__
```
dvdrental=# explain select address from address where postal_code = '90920';
                       QUERY PLAN                        
---------------------------------------------------------
 Seq Scan on address  (cost=0.00..15.54 rows=1 width=20)
   Filter: ((postal_code)::text = '90920'::text)
```
__Создала индекс по полю postal_code:__
```
create index on address(postal_code);
```

2. Прислать текстом результат команды explain, в которой используется данный индекс

__Результат explain с использованием индекса:__
```
dvdrental=# explain select address from address where postal_code = '90920';
                                       QUERY PLAN                                       
-------------------------------------------------------------------------
 Index Scan using address_postal_code_idx on address  (cost=0.28..2.49 rows=1 width=20)
   Index Cond: ((postal_code)::text = '90920'::text)
(2 строки)
```

3. Реализовать индекс для полнотекстового поиска

__Результат explain без использования индекса:__
```
dvdrental=# explain select title, description from film where to_tsvector('english', description) @@ to_tsquery('cat & dog');
                                         QUERY PLAN                                          
-------------------------------------------------------------------------
 Seq Scan on film  (cost=0.00..566.50 rows=1 width=109)
   Filter: (to_tsvector('english'::regconfig, description) @@ to_tsquery('cat & dog'::text))
(2 строки)
```
__Создала индекс GIN по полю 'description':__
```
dvdrental=# create index film_descr_idx on film using gin (to_tsvector('english', description));
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

4. Реализовать индекс на часть таблицы или индекс
на поле с функцией

__Индекс на поле с функцией__
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

__Индекс на часть таблицы__
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

5. Создать индекс на несколько полей
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
