# Домашнее задание Сартисона Евгения №4 #


Высокая доступность: развертывание Patroni

Цель:
развернуть отказоустойчивый кластер PostgreSQL с Patroni;

Описание/Пошаговая инструкция выполнения домашнего задания:
"BananaFlow" вышла на новый рынок — Австралию. В день запуска платформы в Австралии количество запросов к базе данных выросло в 10 раз. База данных не выдержала нагрузки: один из серверов вышел из строя, и система была недоступна в течение нескольких часов.

Поставщик "GreenBanana Co." из Сиднея не получил вовремя информацию о заказе на 10 тонн бананов. В результате, бананы не были доставлены к началу фестиваля "BananaFest", который ежегодно проводится в Сиднее. Организаторы фестиваля были вынуждены закупать бананы у конкурентов по завышенным ценам, а "GreenBanana Co." потеряла контракт на следующий год.

Алексей понял, что нужно срочное решение. Он решил развернуть высокодоступный кластер на основе Patroni, чтобы обеспечить отказоустойчивость и бесперебойную работу базы данных.

Однако на пути к успеху его ждали новые вызовы:
***Как правильно настроить кластер из трёх узлов?***
***Как обеспечить синхронизацию данных между узлами?***
***Как проверить отказоустойчивость системы?***
Алексей знал, что от его действий зависит успех компании: если кластер будет работать стабильно, это позволит избежать сбоев и сохранить репутацию "BananaFlow".


## (1) Создайте 3 виртуальные машины для etcd и 3 виртуальные машины для Patroni.
Для экономии времени и упрощения конфигурирования, создал 3 виртуальные машины для Postgres+Patroni и одну отдельную для HaProxy в YC 
![image](https://github.com/user-attachments/assets/6e6dab44-1d42-4441-80d1-db29bac76294)

IP адреса для созданных VM
| Machine  | Private IP |
| ------------- | ------------- |
| pgnode1  | 192.168.0.24  |
| pgnode2  | 192.168.0.25  |
| pgnode3  | 192.168.0.26  |
| pghaproxy1  | 192.168.0.30  |




## (2) Разверните HA-кластер PostgreSQL с использованием Patroni.

## (2.1) установка\настройка Postgres
pgnode[1-3]:регистраация репозитория postgresql.org
>sudo apt install -y postgresql-common
>sudo /usr/share/postgresql-common/pgdg/apt.postgresql.org.sh

pgnode[1-3]:установка Postgres 17
>sudo
>apt install postgresql-17

pgnode1:смена пароля для схемы postgres и создание пользователей postgres и pgbouncer 

>psql \password --- выставляем пароль postgres

>create user replicator replication login encrypted password 'replicator';

>create user pgbouncer login encrypted password 'pgbouncer';

pgnode[1-3]: правка pg_hba.conf и postgresql.conf, чтобы разрешить удаленные подключения 
![image](https://github.com/user-attachments/assets/8c17094f-5994-4fa4-877f-6387d7432e61)
![image](https://github.com/user-attachments/assets/5ccbba28-b840-4339-90d0-e4192ad4e83d)
>esartison@pgnode3:~$ sudo systemctl restart postgresql

pgnode[2,3]: удаление содержимое каталога pgdata на репликах
>systemctl stop postgresql
>rm -rf /var/lib/postgresql/17/main/*


## (2.2) установка\настройка ETCD

pgnode[1-3]:Установка дистрибутива из под ROOT-а
>cd /tmp && wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz && tar xzvf etcd-v3.5.5-linux-amd64.tar.gz

>sudo mv /tmp/etcd-v3.5.5-linux-amd64/etcd* /usr/local/bin/



pgnode[1-3]: Создание пользователя, настройка директорий и прав
> sudo groupadd --system etcd

> sudo useradd -s /sbin/nologin --system -g etcd etcd

Создаем каталоги etcd, меняем владельца и права:
>mkdir /opt/etcd

>mkdir /etc/etcd

>mkdir /var/lib/etcd

>chown -R etcd:etcd /opt/etcd /var/lib/etcd /etc/etcd

>chmod -R 700 /opt/etcd/ /var/lib/etcd /etc/etcd


pgnode1: правка /etc/etcd/etcd.conf

root@pgnode1:/tmp# cat /etc/etcd/etcd.conf
```
root@pgnode1:/tmp# cat /etc/etcd/etcd.conf
ETCD_NAME="etcd1"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.24:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.24:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.24:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.24:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.0.24:2380,etcd2=http://192.168.0.25:2380,etcd3=http://192.168.0.26:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

pgnode2: правка /etc/etcd/etcd.conf
```
root@pgnode2:/tmp# cat /etc/etcd/etcd.conf
ETCD_NAME="etcd2"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.25:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.25:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.25:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.25:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.0.24:2380,etcd2=http://192.168.0.25:2380,etcd3=http://192.168.0.26:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

pgnode3: правка /etc/etcd/etcd.conf
```
root@pgnode3:/tmp# cat /etc/etcd/etcd.conf
ETCD_NAME="etcd3"
ETCD_LISTEN_CLIENT_URLS="http://192.168.0.26:2379,http://127.0.0.1:2379"
ETCD_ADVERTISE_CLIENT_URLS="http://192.168.0.26:2379"
ETCD_LISTEN_PEER_URLS="http://192.168.0.26:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://192.168.0.26:2380"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-postgres-cluster"
ETCD_INITIAL_CLUSTER="etcd1=http://192.168.0.24:2380,etcd2=http://192.168.0.25:2380,etcd3=http://192.168.0.26:2380"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_ELECTION_TIMEOUT="10000"
ETCD_HEARTBEAT_INTERVAL="2000"
ETCD_INITIAL_ELECTION_TICK_ADVANCE="false"
ETCD_ENABLE_V2="true"
```

pgnode[1-3]: правка /etc/systemd/system/etcd.service
```
root@pgnode3:/tmp# cat  /etc/systemd/system/etcd.service
[Unit]
Description=Etcd Server
Documentation=https://github.com/etcd-io/etcd
After=network.target
After=network-online.target
Wants=network-online.target

[Service]
User=etcd
Type=notify
#WorkingDirectory=/var/lib/etcd/
WorkingDirectory=/opt/etcd/
EnvironmentFile=-/etc/etcd/etcd.conf
User=etcd
# set GOMAXPROCS to number of processors
ExecStart=/bin/bash -c "GOMAXPROCS=$(nproc) /usr/local/bin/etcd"
Restart=on-failure
LimitNOFILE=65536
IOSchedulingClass=realtime
IOSchedulingPriority=0
Nice=-20

[Install]
WantedBy=multi-user.target
```

Настройка автозапуска и проверка состояния
>systemctl daemon-reload
>systemctl enable etcd
>systemctl start etcd
>systemctl status etcd
![image](https://github.com/user-attachments/assets/6f40dd85-85ac-43d9-a139-2880840e6d06)

>etcdctl endpoint health --cluster -w table
![image](https://github.com/user-attachments/assets/7503db04-7f8a-40b7-9931-ea22ec5adc6d)


## (2.3) установка\настройка Patroni

## (3) Настройте HAProxy для балансировки нагрузки.

## (4) Проверьте отказоустойчивость кластера, имитируя сбой на одном из узлов.

## (5) Дополнительно: Настройте бэкапы с использованием WAL-G или pg_probackup.
