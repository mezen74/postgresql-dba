# Установка ПО на виртуальные машины кластера

## Установка репозитория epel (на всех ВМ)
```
sudo yum install epel-release
```

## Установка etcd (только pg-proxy)
```
sudo yum install etcd
sudo vi /etc/etcd/etcd.conf
sudo systemctl start etcd
sudo systemctl enable etcd
```

## Установка HAProxy (только pg-proxy)
```
sudo yum install haproxy
sudo setsebool -P haproxy_connect_any=1
sudo systemctl enable haproxy
```

## Установка confd (только pg-proxy)
```
wget https://github.com/kelseyhightower/confd/releases/download/v0.16.0/confd-0.16.0-linux-amd64
sudo mkdir -p /opt/confd/bin
sudo mv confd-0.16.0-linux-amd64 /opt/confd/bin/confd
sudo chmod +x /opt/confd/bin/confd
export PATH="$PATH:/opt/confd/bin"
sudo mkdir -p /etc/confd/{conf.d,templates}
sudo vi /etc/confd/conf.d/haproxy.toml
sudo vi /etc/confd/templates/haproxy.tmpl
sudo vi /etc/systemd/system/confd.service
sudo vi /etc/confd/confd.toml
sudo systemctl start confd
sudo systemctl enable confd
```


## Установка PostgreSQL (pg-db1, pg-db2)
```
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

sudo yum install -y postgresql15-server
```
__Кластер не инициализировать, сервис не запускать!__

## Установка Patroni (pg-db1, pg-db2)
```
sudo yum install python3-psycopg2 patroni patroni-etcd

sudo vi /etc/patroni/patroni.yml

sudo systemctl start patroni
sudo systemctl status patroni
sudo systemctl enable patroni
patronictl -c /etc/patroni/patroni.yml list

curl http://192.168.2.11:8008/metrics
```

## Установка Prometheus (только pg-proxy)
```
wget https://github.com/prometheus/prometheus/releases/download/v2.49.1/prometheus-2.49.1.linux-amd64.tar.gz
tar -xvf prometheus-2.49.1.linux-amd64.tar.gz
mv prometheus-2.49.1.linux-amd64 prometheus-files
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus
sudo mkdir /var/lib/prometheus
sudo chown -R prometheus:prometheus /var/lib/prometheus/
sudo cp prometheus-files/prometheus /usr/local/bin/
sudo cp prometheus-files/promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus
sudo chown prometheus:prometheus /usr/local/bin/promtool
sudo cp -r prometheus-files/consoles /etc/prometheus
sudo cp -r prometheus-files/console_libraries /etc/prometheus
sudo chown -R prometheus:prometheus /etc/prometheus/consoles
sudo chown -R prometheus:prometheus /etc/prometheus/console_libraries
sudo vi /etc/prometheus/prometheus.yml
sudo vi /etc/systemd/system/prometheus.service
sudo systemctl daemon-reload
sudo systemctl start prometheus
sudo systemctl enable prometheus
```

## Установка Grafana (только pg-proxy)
```
sudo yum install fontconfig
sudo yum install -y https://dl.grafana.com/oss/release/grafana-10.3.1-1.x86_64.rpm
sudo systemctl start grafana-server
```
Подключиться к web-интерфейсу Grafana:порт 3000, admin/admin. Сменить пароль (указала новый пароль Otus135)

В веб-интерфейсе Graphana добавить источник данных Prometheus http://192.168.2.10:9090

Импортировать dashboard id [18870]
(https://grafana.com/grafana/dashboards/18870-postgresql-patroni/)


## Установка sysbench (pg-proxy)
```
curl -s https://packagecloud.io/install/repositories/akopytov/sysbench/script.rpm.sh | sudo bash
sudo yum -y install sysbench
git clone https://github.com/Percona-Lab/sysbench-tpcc
psql -h 192.168.2.10 -p 5000 -U postgres
CREATE DATABASE sbtest;
CREATE USER sbtest WITH PASSWORD 'otus135';
GRANT ALL PRIVILEGES ON DATABASE sbtest TO sbtest;
\c sbtest
grant all on schema public to sbtest;
```
Подготовка таблиц
```
cd sysbench-tpcc/
./tpcc.lua --pgsql-host=192.168.2.10 --pgsql-port=5000 --pgsql-user=sbtest --pgsql-password=otus135 --pgsql-db=sbtest --time=120 --threads=56 --report-interval=1 --tables=1 --scale=10 --use_fk=0  --trx_level=RC --db-driver=pgsql prepare
```


