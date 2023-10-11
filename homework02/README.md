# Домашнее задание #2

## Установка PostgreSQL


- создать ВМ с Ubuntu 20.04/22.04 или развернуть докер любым удобным способом

    **Создала ВМ c Ubuntu 22.04 на ноутбуке (KVM)**
- поставить на нем Docker Engine

    **Подключила репозиторий Docker, установила Docker из пакетов**
- сделать каталог /var/lib/postgres

```
sudo mkdir /var/lib/postgres
```

- развернуть контейнер с PostgreSQL 15 смонтировав в него /var/lib/postgresql
```
sudo docker network create pg-net
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
- развернуть контейнер с клиентом postgres
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
- подключится из контейнера с клиентом к контейнеру с сервером и сделать
таблицу с парой строк

    **Сделала таблицу скриптом из первого ДЗ**

- подключится к контейнеру с сервером с ноутбука/компьютера извне инстансов GCP/ЯО/места установки докера

    Подключилась с ноутбука: 
```
psql -h 192.168.122.177 -p 5432 -U postgres
```
где 192.168.122.177 - адрес ВМ Ubuntu, на которой развёрнут Docker

- удалить контейнер с сервером
```
sudo docker stop pg-server
sudo docker rm pg-server
```
- создать его заново
```
sudo docker run --name pg-server --network pg-net -e POSTGRES_PASSWORD=postgres -d -p 5432:5432 -v /var/lib/postgres:/var/lib/postgresql/data postgres:15
```
- подключится снова из контейнера с клиентом к контейнеру с сервером
```
sudo docker run -it --rm --network pg-net --name pg-client postgres:15 psql -h pg-server -U postgres
```
- проверить, что данные остались на месте

Данные остались на месте:
```
postgres=# select * from persons;
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
  3 | sergey     | sergeev

```
