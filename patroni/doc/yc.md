# Развертывание ВМ в Yandex Cloud

## Создание виртуальной сети

### Создать облачную сеть
```
yc vpc network create \
  --name pg-network \
  --description "Network for lab Patroni"

```

### Создать подсеть в облачной сети
```
yc vpc subnet create \
  --name pg-subnet-b \
  --zone ru-central1-b \
  --range 192.168.2.0/24 \
  --network-name pg-network \
  --description "Network for lab Patroni"

```

## Создать ВМ
### ВМ, на которой будут etcd, HAProxy и Prometheus/Grafana
```
yc compute instance create \
  --name pg-proxy \
  --hostname pg-proxy \
  --network-interface subnet-name=pg-subnet-b,nat-ip-version=ipv4,address=192.168.2.10 \
  --memory 4G \
  --cores 2 \
  --create-boot-disk image-folder-id=standard-images,image-family=centos-7,size=20,auto-delete=true \
  --zone ru-central1-b \
  --ssh-key ~/.ssh/id_ed25519.pub

```
Подключаться: ssh yc-user@внешнийIp


### ВМ для Postgres/Patroni

```
yc compute instance create \
  --name pg-db1 \
  --hostname pg-db1 \
  --network-interface subnet-name=pg-subnet-b,nat-ip-version=ipv4,address=192.168.2.11 \
  --memory 4G \
  --cores 2 \
  --create-boot-disk image-folder-id=standard-images,image-family=centos-7,size=20,auto-delete=true \
  --zone ru-central1-b \
  --ssh-key ~/.ssh/id_ed25519.pub

```

```
yc compute instance create \
  --name pg-db2 \
  --hostname pg-db2 \
  --network-interface subnet-name=pg-subnet-b,nat-ip-version=ipv4,address=192.168.2.12 \
  --memory 4G \
  --cores 2 \
  --create-boot-disk image-folder-id=standard-images,image-family=centos-7,size=20,auto-delete=true \
  --zone ru-central1-b \
  --ssh-key ~/.ssh/id_ed25519.pub

```

### Дополнительная ВМ для проверки добавления нового узла в кластер
```
yc compute instance create \
  --name pg-db3 \
  --hostname pg-db3 \
  --network-interface subnet-name=pg-subnet-b,nat-ip-version=ipv4,address=192.168.2.13 \
  --memory 4G \
  --cores 2 \
  --create-boot-disk image-folder-id=standard-images,image-family=centos-7,size=20,auto-delete=true \
  --zone ru-central1-b \
  --ssh-key ~/.ssh/id_ed25519.pub
```
