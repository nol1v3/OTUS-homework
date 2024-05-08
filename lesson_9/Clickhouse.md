# Настройка кластера Clickhouse.
> Установка выполнена через **сервис:** YandexCloud.

>> Установка будет выполнена из *.deb пакетов по **руководству:** [Clickhouse Ubuntu](https://clickhouse.com/docs/en/install)
>>> Установка кластера будет выполнена по **руководству**: [Clickhouse](https://clickhouse.com/docs/en/architecture/cluster-deployment)  

| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-clickhouse01    | 158.160.131.260   |
| srv-ubu-clickhouse02    | 158.160.131.270   |
| srv-ubu-clickhouse03    | 158.160.131.280   |

## Установка сервера Clickhouse. (На всех серверах)
```bash
sudo apt-get install apt-transport-https ca-certificates dirmngr
sudo apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv 8919F6BD2B48D754
echo "deb https://packages.clickhouse.com/deb stable main" | sudo tee \
    /etc/apt/sources.list.d/clickhouse.list
sudo apt-get update
sudo apt-get install -y clickhouse-server clickhouse-client
sudo service clickhouse-server start
clickhouse-client # or "clickhouse-client --password" if you set up a password.
```

## Конфигурирование сервера srv-ubu-clickhouse01.
> Настройка сети и логирования.
```bash
cd /etc/clickhouse-server/config.d/
vim network-and-logging.xml

<clickhouse>
        <logger>
                <level>debug</level>
                <log>/var/log/clickhouse-server/clickhouse-server.log</log>
                <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
                <size>1000M</size>
                <count>3</count>
        </logger>
        <display_name>clickhouse</display_name>
        <listen_host>0.0.0.0</listen_host>
        <http_port>8123</http_port>
        <tcp_port>9000</tcp_port>
        <interserver_http_port>9009</interserver_http_port>
</clickhouse>
```
> Настройка clickhouse-keeper.
>> ClickHouse Keeper предоставляет систему координации репликации данных и выполнения распределенных DDL-запросов.
```
vim /etc/clickhouse-keeper/keeper_config.xml

<clickhouse>
  <keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>1</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>trace</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>158.160.131.260</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>158.160.131.270</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>158.160.131.280</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
  </keeper_server>
</clickhouse>
```
> Настройка макросов.
>> Макросы cluster, shard и replica уменьшают сложность распределенных DDL команд. Настроенные значения автоматически подставляются в запросы DDL.
```
С ENGINE = ReplicatedReplacingMergeTree('/clickhouse/tables/{shard}/table_name', '{replica}', ver)
НА ENGINE = ReplicatedReplacingMergeTree
```
```
vim macros.xml

<clickhouse>
  <macros>
    <cluster>test_cluster</cluster>
    <shard>1</shard>
    <replica>replica_1</replica>
  </macros>
</clickhouse>
```
> Настройка топологии кластера и шардов.
```bash
vim remote-servers.xml

<clickhouse>
  <remote_servers replace="true">
    <test_cluster>
    <secret>testsecret</secret>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>158.160.131.260</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>158.160.131.270</host>
                <port>9000</port>
            </replica>
        </shard>
    </test_cluster>
  </remote_servers>
</clickhouse>
```
> Файл конфигурации use-keeper.xml настраивает сервер ClickHouse для использования ClickHouse Keeper для координации репликации и распределенного DDL
```bash
vim use-keeper.xml

<clickhouse>
    <zookeeper>
        <node index="1">
            <host>158.160.131.260</host>
            <port>9181</port>
        </node>
        <node index="2">
            <host>158.160.131.270</host>
            <port>9181</port>
        </node>
        <node index="3">
            <host>158.160.131.280</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

## Конфигурирование сервера srv-ubu-clickhouse02.
```bash
cd /etc/clickhouse-server/config.d/
vim network-and-logging.xml

<clickhouse>
        <logger>
                <level>debug</level>
                <log>/var/log/clickhouse-server/clickhouse-server.log</log>
                <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
                <size>1000M</size>
                <count>3</count>
        </logger>
        <display_name>clickhouse</display_name>
        <listen_host>0.0.0.0</listen_host>
        <http_port>8123</http_port>
        <tcp_port>9000</tcp_port>
        <interserver_http_port>9009</interserver_http_port>
</clickhouse>
```
```bash
vim /etc/clickhouse-keeper/keeper_config.xml

<clickhouse>
  <keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>2</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>trace</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>158.160.131.260</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>158.160.131.270</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>158.160.131.280</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
  </keeper_server>
</clickhouse>
```
```bash
vim macros.xml

<clickhouse>
  <macros>
    <cluster>test_cluster</cluster>
    <shard>2</shard>
    <replica>replica_1</replica>
  </macros>
</clickhouse>
```
```bash
vim remote-servers.xml

<clickhouse>
  <remote_servers replace="true">
    <test_cluster>
    <secret>testsecret</secret>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>158.160.131.260</host>
                <port>9000</port>
            </replica>
        </shard>
        <shard>
            <internal_replication>true</internal_replication>
            <replica>
                <host>158.160.131.270</host>
                <port>9000</port>
            </replica>
        </shard>
    </test_cluster>
  </remote_servers>
</clickhouse>
```
```bash
vim use-keeper.xml

<clickhouse>
    <zookeeper>
        <node index="1">
            <host>158.160.131.260</host>
            <port>9181</port>
        </node>
        <node index="2">
            <host>158.160.131.270</host>
            <port>9181</port>
        </node>
        <node index="3">
            <host>158.160.131.280</host>
            <port>9181</port>
        </node>
    </zookeeper>
</clickhouse>
```

## Конфигурирование сервера srv-ubu-clickhouse03.
На srv-ubu-clickhouse03 не будут храниться данные и используется Clickhouse Keeper только для предоставления третьего узла в кворуме.
```bash
cd /etc/clickhouse-server/config.d/
vim network-and-logging.xml

<clickhouse>
        <logger>
                <level>debug</level>
                <log>/var/log/clickhouse-server/clickhouse-server.log</log>
                <errorlog>/var/log/clickhouse-server/clickhouse-server.err.log</errorlog>
                <size>1000M</size>
                <count>3</count>
        </logger>
        <display_name>clickhouse</display_name>
        <listen_host>0.0.0.0</listen_host>
        <http_port>8123</http_port>
        <tcp_port>9000</tcp_port>
        <interserver_http_port>9009</interserver_http_port>
</clickhouse>
```

```bash
vim /etc/clickhouse-keeper/keeper_config.xml

<clickhouse>
  <keeper_server>
    <tcp_port>9181</tcp_port>
    <server_id>3</server_id>
    <log_storage_path>/var/lib/clickhouse/coordination/log</log_storage_path>
    <snapshot_storage_path>/var/lib/clickhouse/coordination/snapshots</snapshot_storage_path>

    <coordination_settings>
        <operation_timeout_ms>10000</operation_timeout_ms>
        <session_timeout_ms>30000</session_timeout_ms>
        <raft_logs_level>trace</raft_logs_level>
    </coordination_settings>

    <raft_configuration>
        <server>
            <id>1</id>
            <hostname>158.160.131.260</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>2</id>
            <hostname>158.160.131.270</hostname>
            <port>9234</port>
        </server>
        <server>
            <id>3</id>
            <hostname>158.160.131.280</hostname>
            <port>9234</port>
        </server>
    </raft_configuration>
  </keeper_server>
</clickhouse>
```

## Проверяем кластер Clickhouse.
>Запускаем Clickhouse и Clickhouse Keeper
```bash
systemctl start clickhouse-server.service
systemctl start clickhouse-keeper.service
```
> В случае ошибок проверить лог-файлы по путям из конфигураций.
> Подключаемся к 158.160.131.260 и проверяем, что настроенный выше  кластер существует.
```bash
clickhouse-client
SHOW CLUSTERS

┌─cluster──────┐
│ test_cluster │
└──────────────┘
```
> Создать базу данных в кластере.
```
CREATE DATABASE db1 ON CLUSTER test_cluster;

┌─host────────────┬─port─┬─status─┬─error─┬─num_hosts_remaining─┬─num_hosts_active─┐
│ 158.160.131.270 │ 9000 │      0 │       │                   1 │                0 │
│ 158.160.131.260 │ 9000 │      0 │       │                   0 │                0 │
└─────────────────┴──────┴────────┴───────┴─────────────────────┴──────────────────┘
```
> Создаем таблицу с помощью механизма таблиц MergeTree в кластере и вставляем данные.
```
CREATE TABLE db1.table1 ON CLUSTER cluster_2S_1R
(
    `id` UInt64,
    `column1` String
)
ENGINE = MergeTree
ORDER BY id

INSERT INTO db1.table1 (id, column1) VALUES (1, 'abc');
```
> Подключиться 158.160.131.270 и вставляем строку.
```
INSERT INTO db1.table1 (id, column1) VALUES (2, 'def');
```
> Создайте распределенную таблицу для запроса обоих сегментов на обоих узлах.
```
CREATE TABLE db1.table1_dist ON CLUSTER test_cluster
(
    `id` UInt64,
    `column1` String
)
ENGINE = Distributed('test_cluster', 'db1', 'table1', rand())
```
Подключитесь к любому из 158.160.131.260 или 158.160.131.270 и запросим распределенную таблицу, чтобы просмотреть обе строки.
```
SELECT * FROM db1.table1_dist;

┌─id─┬─column1─┐
│  2 │ def     │
└────┴─────────┘
┌─id─┬─column1─┐
│  1 │ abc     │
└────┴─────────┘
```
Благодаря кластеру мы можем грузить данный в 2 таблицы на 2 узлах, что означает прирост скорости записи в 2 раза и.т.д.
