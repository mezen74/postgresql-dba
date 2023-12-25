# Домашнее задание #9

## Бэкапы


1. Создаем ВМ/докер c ПГ.
2. Создаем БД, схему и в ней таблицу.
```
create database lab_backup;
\c lab_backup 
create schema otus;
set search_path to otus,public;
create table customer (customer_id integer primary key, customer_name char(100) unique not null);
```

3. Заполним таблицы автосгенерированными 100 записями.
```
insert into customer (customer_id, customer_name) SELECT generate_series(1,100), md5(random()::text);
```
4. Под линукс пользователем Postgres создадим каталог для бэкапов
```
sudo mkdir -p /data/backup;
sudo chown postgres:postgres /data/backup
```

5. Сделаем логический бэкап используя утилиту COPY
```
copy customer to '/data/backup/customer.sql';
```

6. Восстановим в 2 таблицу данные из бэкапа.
```
create table customer_new (like customer including all);
copy customer_new from '/data/backup/customer.sql';
```

7. Используя утилиту pg_dump создадим бэкап в кастомном сжатом формате двух таблиц
```
pg_dump -d lab_backup -U postgres -Fc -n otus -t 'otus.customer*' >/data/backup/lab_backup.dump
```

8. Используя утилиту pg_restore восстановим в новую БД только вторую таблицу

```
create database lab_backup2;
create schema otus;
```

__При использовании -t, pg_dump не выгружает прочие объекты, от которых выгружаемые таблицы могут зависеть. Поэтому схему 'otus' в новой базе потребовалось создать вручную.__

__Сначала выполнила восстановление так:__
```
pg_restore -d lab_backup2 -U postgres  -t 'otus.customer_new'  /data/backup/lab_backup.dump
```
__После восстановления обнаружила, что не были восстановлены ограничения целостности primary key и unique (я специально создала таблицу с ограничениями целостности, чтобы проверить, как они будут восстановлены). Обнаружила в документации на pg_restore такое примечание: "И хотя pg_dump с флагом -t также выгружает подчинённые объекты (например, индексы) выбранных таблиц, pg_restore с флагом -t такие подчинённые объекты не обрабатывает."__ 

__Воспользовалась флагами -l и -L:__

Вывела оглавление архива в файл:

```
pg_restore -l /data/backup/lab_backup.dump >/data/backup/lab_backup.list
```

Отредактировала файл - оставила только объекты, которые относятся к таблице customer_new:

```
217; 1259 24780 TABLE otus customer_new postgres
3338; 0 24780 TABLE DATA otus customer_new postgres
3191; 2606 24786 CONSTRAINT otus customer_new customer_new_customer_name_key postgres
3193; 2606 24784 CONSTRAINT otus customer_new customer_new_pkey postgres
```

Выполнила восстановление:

```
pg_restore -d lab_backup2 -U postgres -L /data/backup/lab_backup2.list  /data/backup/lab_backup.dump
```

__При таком способе в новую базу была восстановлена только вторая таблица, как и требовалось в задании, при этом ограничения целостности также были восстановлены.__
