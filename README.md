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
У нас etcd и postgres будут бежать на 3 узлах, а на еще одном будет выполняться HaProxy

***Как обеспечить синхронизацию данных между узлами?***
-- Синхронизация обеспечивается через репликацию, в нашем случае все за нас сделает Patroni

***Как проверить отказоустойчивость системы?***
-- Нужно принудительно остановить один из серверов, имитируя аварию и убедиться что праймари инстанс переехал на другую ноду

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
![image](https://github.com/user-attachments/assets/03d24782-7b94-41d3-8e5d-cf09ab68977e)

![image](https://github.com/user-attachments/assets/5ccbba28-b840-4339-90d0-e4192ad4e83d)
>esartison@pgnode3:~$ sudo systemctl restart postgresql

pgnode[2,3]: удаление содержимое каталога pgdata на репликах
>systemctl stop postgresql
>rm -rf /var/lib/postgresql/17/main/*
Данные будут взяты с мастера

## (2.2) установка\настройка ETCD

Устанавливаем ETCD на те же узлы где и postgres

pgnode[1-3]:Установка дистрибутива из под ROOT-а
>cd /tmp && wget https://github.com/etcd-io/etcd/releases/download/v3.5.5/etcd-v3.5.5-linux-amd64.tar.gz && tar xzvf etcd-v3.5.5-linux-amd64.tar.gz

>sudo mv /tmp/etcd-v3.5.5-linux-amd64/etcd* /usr/local/bin/

pgnode[1-3]: Создание пользователя, настройка директорий и прав
> sudo groupadd --system etcd

> sudo useradd -s /sbin/nologin --system -g etcd etcd

Создание каталогов etcd, меняем владельца и права:
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

pgnode[1-3]: Устанавка Python пакетов
>sudo apt -y install python3 python3-pip python3-dev python3-psycopg2 libpq-dev
>sudo apt install patroni
>pip3 install psycopg2 --break-system-packages
>pip3 install psycopg2-binary --break-system-packages
>pip3 install patroni --break-system-packages
>pip3 install python-etcd --break-system-packages

pgnode1: Создание конфига Patroni /etc/patroni/patroni.yml
>mkdir /etc/patroni/
```
root@pgnode1:/tmp# cat /etc/patroni/patroni.yml
scope: postgres-cluster
name: pgnode1
namespace: /service/

restapi:
  listen: 192.168.0.24:8008 # разное значение на всех узлах
  connect_address: 192.168.0.24:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'password'

etcd:
  hosts: 192.168.0.24:2379, 192.168.0.25:2379, 192.168.0.26:2379

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.0.24,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.24:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```

pgnode2: Создание конфига Patroni /etc/patroni/patroni.yml
>mkdir /etc/patroni/
```
postgres@pgnode2:~$ cat /etc/patroni/patroni.yml
scope: postgres-cluster # одинаковое значение на всех узлах
name: pgnode2 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах

restapi:
  listen: 192.168.0.25:8008 # разное значение на всех узлах
  connect_address: 192.168.0.25:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'password'

etcd:
  hosts: 192.168.0.24:2379, 192.168.0.25:2379, 192.168.0.26:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.0.25,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.25:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: password
    superuser:
      username: postgres
      password: password
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)

```

pgnode3: Создание конфига Patroni /etc/patroni/patroni.yml
>mkdir /etc/patroni/
```
postgres@pgnode3:~$  cat /etc/patroni/patroni.yml
scope: postgres-cluster # одинаковое значение на всех узлах
name: pgnode3 # разное значение на всех узлах
namespace: /service/ # одинаковое значение на всех узлах

restapi:
  listen: 192.168.0.26:8008 # разное значение на всех узлах
  connect_address: 192.168.0.26:8008 # разное значение на всех узлах
  authentication:
    username: patroni
    password: 'password'

etcd:
  hosts: 192.168.0.24:2379, 192.168.0.25:2379, 192.168.0.26:2379 # список всех узлов, на которых установлен etcd

bootstrap:
  method: initdb
  dcs:
    ttl: 60
    loop_wait: 10
    retry_timeout: 27
    maximum_lag_on_failover: 2048576
    master_start_timeout: 300
    synchronous_mode: true
    synchronous_mode_strict: false
    synchronous_node_count: 1
    # standby_cluster:
      # host: 127.0.0.1
      # port: 1111
      # primary_slot_name: patroni
    postgresql:
      use_pg_rewind: false
      use_slots: true
      parameters:
        max_connections: 800
        superuser_reserved_connections: 5
        max_locks_per_transaction: 64
        max_prepared_transactions: 0
        huge_pages: try
        shared_buffers: 512MB
        work_mem: 128MB
        maintenance_work_mem: 256MB
        effective_cache_size: 4GB
        checkpoint_timeout: 15min
        checkpoint_completion_target: 0.9
        min_wal_size: 2GB
        max_wal_size: 4GB
        wal_buffers: 32MB
        default_statistics_target: 1000
        seq_page_cost: 1
        random_page_cost: 4
        effective_io_concurrency: 2
        synchronous_commit: on
        autovacuum: on
        autovacuum_max_workers: 5
        autovacuum_vacuum_scale_factor: 0.01
        autovacuum_analyze_scale_factor: 0.02
        autovacuum_vacuum_cost_limit: 200
        autovacuum_vacuum_cost_delay: 20
        autovacuum_naptime: 1s
        max_files_per_process: 4096
        archive_mode: on
        archive_timeout: 1800s
        archive_command: cd .
        wal_level: replica
        wal_keep_segments: 130
        max_wal_senders: 10
        max_replication_slots: 10
        hot_standby: on
        hot_standby_feedback: True
        wal_log_hints: on
        shared_preload_libraries: pg_stat_statements,auto_explain
        pg_stat_statements.max: 10000
        pg_stat_statements.track: all
        pg_stat_statements.save: off
        auto_explain.log_min_duration: 10s
        auto_explain.log_analyze: true
        auto_explain.log_buffers: true
        auto_explain.log_timing: false
        auto_explain.log_triggers: true
        auto_explain.log_verbose: true
        auto_explain.log_nested_statements: true
        track_io_timing: on
        log_lock_waits: on
        log_temp_files: 0
        track_activities: on
        track_counts: on
        track_functions: all
        log_checkpoints: on
        logging_collector: on
        log_statement: mod
        log_truncate_on_rotation: on
        log_rotation_age: 1d
        log_rotation_size: 0
        log_line_prefix: '%m [%p] %q%u@%d '
        log_filename: 'postgresql-%a.log'
        log_directory: /var/log/postgresql

  initdb:  # List options to be passed on to initdb
    - encoding: UTF8
    - locale: en_US.UTF-8
    - data-checksums

  pg_hba:  # должен содержать адреса ВСЕХ машин, используемых в кластере
    - host all all 0.0.0.0/0 scram-sha-256
    - host replication replicator scram-sha-256

postgresql:
  listen: 192.168.0.26,127.0.0.1:5432 # разное значение на всех узлах
  connect_address: 192.168.0.26:5432 # разное значение на всех узлах
  use_unix_socket: true
  data_dir: /var/lib/postgresql/17/main
  bin_dir: /usr/lib/postgresql/17/bin
  config_dir: /etc/postgresql/17/main
  pgpass: /var/lib/postgresql/.pgpass_patroni
  authentication:
    replication:
      username: replicator
      password: replicator
    superuser:
      username: postgres
      password: postgres
  parameters:
    unix_socket_directories: /var/run/postgresql
    stats_temp_directory: /var/lib/pgsql_stats_tmp

  remove_data_directory_on_rewind_failure: false
  remove_data_directory_on_diverged_timelines: false

#  callbacks:
#    on_start:
#    on_stop:
#    on_restart:
#    on_reload:
#    on_role_change:

  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'

watchdog:
  mode: off  # Allowed values: off, automatic, required
  device: /dev/watchdog
  safety_margin: 5

tags:
  nofailover: false
  noloadbalance: false
  clonefrom: false
  nosync: false

  # specify a node to replicate from (cascading replication)
#  replicatefrom: (node name)
```

pgnode[1-3]:Назначение прав каждой ноде
>chown postgres:postgres -R /etc/patroni

>chmod 700 /etc/patroni

>mkdir /var/lib/pgsql_stats_tmp

>chown postgres:postgres /var/lib/pgsql_stats_tmp


pgnode1:Валидация установки
>sudo -u postgres patroni /etc/patroni/patroni.yml
![image](https://github.com/user-attachments/assets/9a2cf49d-9175-4ff7-a92f-6dd12ce89ff6)

pgnode2:Валидация установки
![image](https://github.com/user-attachments/assets/b1ab956f-bf8f-4657-b47b-903523d91ea2)

pgnode3:Валидация установки
![image](https://github.com/user-attachments/assets/c64ade65-d4b7-438a-bcb0-57b64a37fc68)

pgnode[1-3]: добавить службу Patroni в автозапуск
```
root@pgnode1:/tmp# cat /etc/systemd/system/patroni.service
[Unit]
Description=High availability PostgreSQL Cluster
After=syslog.target network.target

[Service]
Type=simple
User=postgres
Group=postgres

# Read in configuration file if it exists, otherwise proceed
EnvironmentFile=-/etc/patroni_env.conf

# Start the patroni process
ExecStart=/usr/bin/patroni /etc/patroni/patroni.yml

# Send HUP to reload from patroni.yml
ExecReload=/bin/kill -s HUP $MAINPID

# only kill the patroni process, not it's children, so it will gracefully stop postgres
KillMode=process

# Give a reasonable amount of time for the server to start up/shut down
TimeoutSec=60

# Do not restart the service if it crashes, we want to manually inspect database on failure
Restart=no

[Install]
WantedBy=multi-user.target
```
>systemctl daemon-reload
>systemctl enable patroni
>systemctl start patroni
>systemctl status patroni
![image](https://github.com/user-attachments/assets/c4200e03-f9f3-4c1e-a623-4f576192775a)


Просмотреть состояние кластера можно командой
>patronictl -c /etc/patroni/patroni.yml list

![image](https://github.com/user-attachments/assets/5923d840-1974-4c39-adba-7216bb25c29e)

Все отработало как и ожидали

## (2.4) установка\настройка PgBouncer
pgnode[1-3]: Установка PGBouncer
>apt -y install pgbouncer


pgnode[1-3]:Подготовка конфига  /etc/pgbouncer/pgbouncer.ini
```
root@pgnode3:/tmp# cat  /etc/pgbouncer/pgbouncer.ini
[databases]
postgres = host=127.0.0.1 port=5432 dbname=postgres
* = host=127.0.0.1 port=5432

[pgbouncer]
logfile = /var/log/postgresql/pgbouncer.log
pidfile = /var/run/postgresql/pgbouncer.pid
listen_addr = *
listen_port = 6432
unix_socket_dir = /var/run/postgresql
auth_type = md5
#auth_type = trust
auth_file = /etc/pgbouncer/userlist.txt
auth_user = postgres
auth_query = SELECT usename, passwd FROM pg_shadow WHERE usename=$1
#admin_users = pgbouncer, postgres
admin_users = postgres
ignore_startup_parameters = extra_float_digits,geqo,search_path

pool_mode = session
#pool_mode = transaction
server_reset_query = DISCARD ALL
max_client_conn = 10000
#default_pool_size = 20
reserve_pool_size = 1
reserve_pool_timeout = 1
max_db_connections = 1000
#max_client_conn = 900
default_pool_size = 500
pkt_buf = 8192
listen_backlog = 4096
log_connections = 1 
log_disconnections = 1
```

Создание userlist.txt со списком пользователей для работы через PGBouncer:
```
root@pgnode3:/tmp# cat /etc/pgbouncer/userlist.txt
"postgres" "postgres"
"pgbouncer" "pgbouncer"
"replicator" "replicator"
```

Перезапуск PGBouncer
>systemctl restart pgbouncer

![image](https://github.com/user-attachments/assets/b8f50046-d4ea-4982-8cb7-1dc754da908b)
Сервис работоспособен

Проверка подключения к Postgres через PGBouncer (порт 6432):
>psql -p 6432 -h 127.0.0.1 -U postgres postgres

![image](https://github.com/user-attachments/assets/eaeb4bbf-e302-4104-a933-87d828375a87)
Все прошло успешно


## (3) Настройте HAProxy для балансировки нагрузки.
pghaproxy1: Устанавливка HAProxy:
>apt -y install haproxy
делаем только на выделенной машине pghaproxy1

pghaproxy1: Создание нового файла конфигурации  /etc/haproxy/haproxy.cfg
```
root@pghaproxy1:/home/esartison# cat /etc/haproxy/haproxy.cfg
global

        maxconn 10000
        log     127.0.0.1 local2

defaults
        log global
        mode tcp
        retries 2
        timeout client 30m
        timeout connect 4s
        timeout server 30m
        timeout check 5s

listen stats
    mode http
    bind *:7000
    stats enable
    stats uri /

listen postgres
    bind *:7432
    option httpchk
    http-check expect status 200
    default-server inter 3s fall 3 rise 2 on-marked-down shutdown-sessions
    server node1 192.168.0.24:6432 maxconn 10000 check port 8008
    server node2 192.168.0.25:6432 maxconn 100 check port 8008
    server node3 192.168.0.26:6432 maxconn 100 check port 8008
```

pghaproxy1: Перезагрузка HAProxy:
>sudo systemctl restart haproxy

pghaproxy1: Проверка работоспособности
>sudo systemctl status haproxy
![image](https://github.com/user-attachments/assets/a10dddef-57cb-4175-940e-019aa8850181)
Сервис рабочий


Проверка подключения к HAProxy по порту 7432
![image](https://github.com/user-attachments/assets/2da98e66-2ebf-4a91-9c5c-fd26c0eb66a3)
Успешно подключился с локальной Win машины

## (4) Проверьте отказоустойчивость кластера, имитируя сбой на одном из узлов.

проверка текущей конфигурации
>patronictl -c /etc/patroni/patroni.yml list

![image](https://github.com/user-attachments/assets/55e76b71-7a92-4b61-895e-7e6b8f4363e9)


принудительно остановил первую ноду
![image](https://github.com/user-attachments/assets/f000d117-4db0-40ad-ab41-221b982b9fa8)

Лидер переключился на третью ноду
![image](https://github.com/user-attachments/assets/2bb8b8af-111c-483b-90e4-6ffc480fa446)

Все отработало!

## (5) Дополнительно: Настройте бэкапы с использованием WAL-G или pg_probackup.

создание файлового хранилища для хранения бэкапов в YC 
![image](https://github.com/user-attachments/assets/b9705fbf-bdf2-45d2-8435-a2e5badbd598)

pgnodeх[1-3]: Создание директории для бэкапов
 >mkdir /backup
 >sudo mount -t virtiofs pgbackup /backup
 >chown postgres:postgres /backup
![image](https://github.com/user-attachments/assets/42c6dbd2-d959-490d-9b09-82e665530147)
Примонтировал диск на все 3 ноды

pg_probackup не поддерживается для 17й версии Postgres
![image](https://github.com/user-attachments/assets/5b15856b-afb8-442d-84bc-e91bd8b43b8f)
![image](https://github.com/user-attachments/assets/564cf8f4-5985-406e-b73c-cebc6f2e9bb3)

Нужно тогда настраивать WAL-G.

Установка
>curl -L "https://github.com/wal-g/wal-g/releases/download/v3.0.7/wal-g-pg-ubuntu-20.04-amd64.tar.gz" -o "wal-g.linux-amd64.tar.gz"
>tar -xzf wal-g.linux-amd64.tar.gz
>mv wal-g-pg-ubuntu-20.04-aarch64 /usr/local/bin/wal-g


Подготовка .walg.json файла
{
    "WALG_COMPRESSION_METHOD": "brotli",
    "WALG_DELTA_MAX_STEPS": "5",
    "WALG_FILE_PREFIX": "/backup",
    "PGDATA": "/var/lib/postgresql/17/main",
    "PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}

Подготовка параметров
```
echo "wal_level=replica" >> /etc/postgresql/17/main/postgresql.conf
echo "archive_mode=on" >> /etc/postgresql/17/main/postgresql.conf
echo "archive_command='/usr/local/bin/wal-g wal-push \"%p\" >> /var/log/postgresql/archive_command.log 2>&1' " >> /etc/postgresql/17/main/postgresql.conf
echo "archive_timeout=60" >> /etc/postgresql/17/main/postgresql.conf
echo "restore_command='/usr/local/bin/wal-g wal-fetch \"%f\" \"%p\" >> /var/log/postgresql/restore_command.log 2>&1' " >> /etc/postgresql/17/main/postgresql.conf

killall -s HUP postgres
```

Запуск бэкапа
>  /usr/local/bin/wal-g backup-push /var/lib/postgresql/17/main

Не хватило времени на более сложные вещи..




