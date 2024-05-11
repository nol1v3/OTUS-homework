# Установка сервера apache-cassandra.
> Установка выполнена через **сервис:** YandexCloud.
>> Установка будет выполнена из *.deb пакетов по **руководству:** [CouchDB Ubuntu](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/install/installTarball.html)
>>> Установка кластера будет выполнена по **руководству**: [Cassandra](https://docs.datastax.com/en/cassandra-oss/3.0/cassandra/initialize/initSingleDS.html)  

# Параметры ОС.
```bash
vim /etc/security/limits.conf

cassandra soft nofile 64000
cassandra hard nofile 64000
cassandra soft nproc 64000
cassandra hard nproc 64000
cassandra soft as unlimited
cassandra hard as unlimited
cassandra soft memlock unlimited
cassandra hard memlock unlimited
```

```bash
vim /etc/sysctl.conf

vm.max_map_count = 1048575
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 16777216
net.core.wmem_default = 16777216
net.core.optmem_max = 40960
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
```

# Создание пользователя.
```bash
useradd --user-group --shell=/bin/bash --home-dir=/opt/cassandra --create-home cassandra
groupadd cassandra
usermod -aG cassandra cassandra
```
# Переменные окружения пользователя.
```bash
cat $HOME/.profile

export JAVA_HOME=/usr/lib64/jvm/jre-1.8.0-openjdk
export CASSANDRA_HOME=/opt/cassandra
export CASSANDRA_DATA_HOME=/u01/cassandra/apache-cassandra-4.0.6/data
export CASSANDRA_LOG_DIR=/u01/cassandra/apache-cassandra-4.0.6/logs
export PATH=$PATH:$HOME/bin:$JAVA_HOME/bin:/usr/bin/usr/local/bin:$CASSANDRA_HOME/apache-cassandra-4.0.6/bin
```
# Копирование дистрибутива.
```bash
curl -OL https://downloads.apache.org/cassandra/4.0.6/apache-cassandra-4.0.6-bin.tar.gz
mkdir -p /opt/cassandra/
tar -xzvf apache-cassandra-4.0.6-bin.tar.gz -C /opt/cassandra/
chown cassandra:cassandra -R /opt/cassandra
```
# Создание сервиса systemd
```bash
vim /etc/systemd/system/cassandra.service

[Unit]
Description=Cassandra Database Service
After=network-online.target
Requires=network-online.target

[Service]
User=cassandra
Group=cassandra
ExecStart=/opt/cassandra/apache-cassandra-4.0.6/bin/cassandra -f

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable cassandra.service
```

# Конфигурация сервера apache-cassandra.
Ниже приведены параметры которые необходимо изменить для работы сервера.


Остальные параметры оставить в базовых значениях. (Выполнить для всех 3 серверов listen_address/rpc_address задать соответствующий ip)
```bash
vim /opt/cassandra/apache-cassandra-4.0.6/conf/cassandra.yaml

cluster_name: 'Test Cassandra Cluster'
num_tokens: 16
hints_directory: /u01/cassandra/apache-cassandra-4.0.6/data/hints
authenticator: PasswordAuthenticator
authorizer: CassandraAuthorizer
data_file_directories:
    - /u01/cassandra/apache-cassandra-4.0.6/data
commitlog_directory: /u01/cassandra/apache-cassandra-4.0.6/data/commitlog
cdc_raw_directory: /u01/cassandra/apache-cassandra-4.0.6/data/cdc_raw
saved_caches_directory: /u01/cassandra/apache-cassandra-4.0.6/data/saved_caches
seed_provider:
    - class_name: org.apache.cassandra.locator.SimpleSeedProvider
      parameters:
          - seeds: "10.0.54.17"
listen_address: 10.0.54.17
rpc_address: 10.0.54.17

cat /opt/cassandra/apache-cassandra-4.0.6/conf/cassandra-rackdc.properties
dc=dc1
rack=rack1 
```

# Логирование сервера.
```bash
vim cassandra-env.sh

if [ "x$CASSANDRA_LOG_DIR" = "x" ] ; then
#    CASSANDRA_LOG_DIR="$CASSANDRA_HOME/logs"
    CASSANDRA_LOG_DIR="/u01/cassandra/apache-cassandra-4.0.6/logs/"
fi
```

# Запуск сервера.
```bash
mkdir -p /u01/cassandra/apache-cassandra-4.0.6/data
chown cassandra:cassandra -R /u01/cassandra

systemctl start cassandra
systemctl status cassandra
```

# cqlsh
```bash
cqlsh -u cassandra -p cassandra 

cp -rv /opt/cassandra/apache-cassandra-4.0.6/conf/credentials.sample /home/cassandra/.cassandra/credentials
vim /home/cassandra/.cassandra/credentials

; Licensed to the Apache Software Foundation (ASF) under one
; or more contributor license agreements.  See the NOTICE file
; distributed with this work for additional information
; regarding copyright ownership.  The ASF licenses this file
; to you under the Apache License, Version 2.0 (the
; "License"); you may not use this file except in compliance
; with the License.  You may obtain a copy of the License at
;
;   http://www.apache.org/licenses/LICENSE-2.0
;
; Unless required by applicable law or agreed to in writing,
; software distributed under the License is distributed on an
; "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
; KIND, either express or implied.  See the License for the
; specific language governing permissions and limitations
; under the License.
;
; Sample ~/.cassandra/credentials file.
;
; Please ensure this file is owned by the user and is not readable by group and other users

[PlainTextAuthProvider]
username = cassandra
password = cassandra
```

# nodetool
```bash
nodetool -help
nodetool status 10.0.54.17

vim /opt/cassandra/apache-cassandra-4.0.6/conf/cassandra-env.sh

# Cassandra ships with JMX accessible *only* from localhost.
# To enable remote JMX connections, uncomment lines below
# with authentication and/or ssl enabled. See https://wiki.apache.org/cassandra/JmxSecurity
#
if [ "x$LOCAL_JMX" = "x" ]; then
    LOCAL_JMX=no
fi
...
## Basic file based authn & authz
JVM_OPTS="$JVM_OPTS -Dcom.sun.management.jmxremote.password.file=/opt/cassandra/apache-cassandra-4.0.6/conf/jmxremote.password"

mkdir -p /opt/cassandra/apache-cassandra-4.0.6/conf/
vim /opt/cassandra/apache-cassandra-4.0.6/conf/jmxremote.password

cassandra cassandra
```

# nodetool-скрипт для авторизации пользователей.
```bash
#!/bin/sh
set -e
set -u

password_file=${PASSWORD_FILE:-/opt/cassandra/apache-cassandra-4.0.6/conf/jmxremote.password}
wrap_args="--password-file $password_file"

case "$@" in
  -pwf*|--password-file*)
    wrap_args=""
  ;;
esac

exec nodetool $wrap_args "$@"
```

# Cоздание пользователя в базе данных.
```sql
CREATE ROLE test_role WITH PASSWORD = 'test_role' AND LOGIN = true;
 
CREATE KEYSPACE IF NOT EXISTS test_role
    WITH REPLICATION = { 'class' : 'SimpleStrategy', 'replication_factor' : 3 };
 
GRANT ALL ON KEYSPACE test_keyspace TO test_role;

use test_keyspace;

CREATE TABLE employees(
   id int PRIMARY KEY,
   name text,
   city text,
   salary varint,
   phone varint 
);

# Здесь ((id), city) будет использоваться в качестве ключа раздела и идентификатора для ключа кластеризации.
CREATE TABLE employees_cluster(
   id int,
   name text,
   city text,
   salary varint,
   phone varint
   PRIMARY KEY((id), city) 
);

INSERT INTO employees_cluster 
(id, city, name, phone, salary) 
VALUES ( 2, 'Dhaka', 'Sarlar', 29345345, 3454 );

INSERT INTO employees_cluster 
(id, city, name, phone, salary) 
VALUES ( 3, 'Rangpur', 'Hasan', 01243545, 35345 );

UPDATE employees 
SET
    city = 'Gazipur',
    salary = 12432
WHERE id = 3;

SELECT city, name FROM employees_cluster WHERE id = 2;
SELECT * FROM employees_cluster WHERE id > 2 ALLOW FILTERING;

# Вторичный индекс (дает возможность выполнять поиск по номеру телефона)
CREATE INDEX IF NOT EXISTS ON employees_cluster (phone);

SELECT * FROM employees_cluster WHERE phone = '29345345'
```

