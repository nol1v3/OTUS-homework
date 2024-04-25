# Настройка кластера CouchDB.

> Установка выполнена через **сервис:** YandexCloud.
>> Установка будет выполнена из *.deb пакетов по **руководству:** [CouchDB Ubuntu](https://docs.couchdb.org/en/stable/install/unix.html#installing)
>>> Установка кластера будет выполнена по **руководству**: [CouchDB](https://docs.couchdb.org/en/stable/setup/cluster.html)  


| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-couchdb01    | 158.160.131.243   |
| srv-ubu-couchdb02    | 158.160.131.241   |
| srv-ubu-couchdb03    | 158.160.131.246   |

## Настройка серверов CouchDB. (На всех серверах)
> Базовая настройка параметров ОС для CouchDB.   
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never

sudo vim /etc/default/grub

# Добавляем строку

GRUB_CMDLINE_LINUX="transparent_hugepage=never"

sudo grub-mkconfig -o /boot/grub/grub.cfg
```
> Согласно руководству необходимо выставить **лимиты ОС:**
```bash
sudo vim /etc/security/limits.conf

# Добавляем строки
  
# CouchDB limits
couchdb soft nofile 64000
couchdb hard nofile 64000
couchdb soft nproc 64000 
couchdb hard nproc 64000
couchdb soft fsize unlimited
couchdb hard fsize unlimited
couchdb soft cpu unlimited
couchdb hard cpu unlimited
couchdb soft as unlimited
couchdb hard as unlimited
couchdb soft memlock unlimited
couchdb hard memlock unlimited
```

```bash
# Проверяем режим избыточного выделения памяти
cat /proc/sys/vm/overcommit_memory
# Устанавливаем режим избыточного выделения памяти
vim /etc/sysctl.conf

# Добавить строку

vm.overcommit_memory = 2
vm.overcommit_ratio = 100
```
> Для применения параметров необходимо выполнить **перезагрузку системы**.

## Установка сервера CouchDB. (На всех серверах)
```bash
sudo apt update && sudo apt install -y curl apt-transport-https gnupg
curl https://couchdb.apache.org/repo/keys.asc | gpg --dearmor | sudo tee /usr/share/keyrings/couchdb-archive-keyring.gpg >/dev/null 2>&1
source /etc/os-release
echo "deb [signed-by=/usr/share/keyrings/couchdb-archive-keyring.gpg] https://apache.jfrog.io/artifactory/couchdb-deb/ ${VERSION_CODENAME} main" \
    | sudo tee /etc/apt/sources.list.d/couchdb.list >/dev/null
```
```bash
sudo apt update
sudo apt install -y couchdb
```
> Установка Debian/Ubuntu может быть предварительно настроена для установки на одном узле или в кластере.
>> В кластерной установки по-прежнему необходимо объединить несколько узлов и последовательно настроить каждый сервер.

## Настройка кластера CouchDB.
> Используем предварительно преднастроеный кластер.
>> Для этого в псевдографигеском меню выбираем cluster и заполняем необходимые поля.
>>> После установки проверяем конфиг-файлы и перезапускаем кластер.
```bash
systemctl stop couchdb
cd /opt/couchdb/etc

# Проверяем что сервер принимает соединение по всем ip-адресам
cat default.d/10-bind-address.ini
[chttpd]
bind_address = 0.0.0.0

# Задаем пароль администратору базы данных.
cat local.d/10-admins.ini
[admins]
admin = admin

# Задаем переменные Erlang (используется для поиска серверов CouchDB)
cat vm.args
# ip для каждого сервера свой
-name couchdb@158.160.131.243 
-setcookie 'couchdbcluster'
-kernel inet_dist_use_interface '{0,0,0,0}'
-kernel inet_dist_listen_min 9100
-kernel inet_dist_listen_max 9200

# Устанавливаем количество шардов и реплик шардов.
cat default.ini
[cluster]
q=2
n=3

systemctl start couchdb
systemctl status couchdb
```
> Сервер должен запуститься без ошибок, если сервер упал необходимо посмотреть **лог-файл:** /opt/couchdb/var/log/couchdb.log
>> Остальные метрики можно оставить по умолчанию.

## Проверяем связность серверов CouchDB.
```bash
apt install erlang-base

erl -name couchdb1@158.160.131.243 -setcookie 'couchdbcluster' -kernel inet_dist_listen_min 9100 -kernel inet_dist_listen_max 9200
net_kernel:connect_node('couchdb@158.160.131.241').
true
```
> Проверям от каждого сервера до каждого сервера.
>> Если ответ false то неправильно настроен файл: vm.args или нет сетевого взаимодействия между серверами.

## Объединяем сервера CouchDB в кластер.
> Создаем необходимые базы данных, если они не созданы автоматически. (можно на одном сервере будут отреплецированы)
```bash
curl -X PUT http://admin:admin@127.0.0.1:5984/_users
{"ok":true}
curl -X PUT http://admin:admin@127.0.0.1:5984/_replicator
{"ok":true}
curl -X PUT http://admin:admin@127.0.0.1:5984/_global_changes
{"ok":true}

# Инициализируем кластер на каждом сервере выполняем
curl -X POST -H "Content-Type: application/json" http://admin:admin@127.0.0.1:5984/_cluster_setup -d '{"action": "enable_cluste r", "bind_address":"0.0.0.0", "username": "admin", "password":"password", "node_count":"3"}'
{"ok":true}

# Добавлям сервера к кластеру
curl -X POST -H "Content-Type: application/json" http://admin:admin@158.160.131.243:5984/_cluster_setup -d '{"action": "enable_ cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"admin", "port": 5984, "node_count": "3", "remote_node": "158.160.131.241", "remote_current_user": " admin", "remote_current_password": "admin" }'
{"ok":true}
curl -X POST -H "Content-Type: application/json" http://admin:admin@158.160.131.243:5984/_cluster_setup -d '{"action": "add_node", "host":"158.160.131.241", "port": 5984, "username": "admin", "password":"admin"}'
{"ok":true}

curl -X POST -H "Content-Type: application/json" http://admin:admin@158.160.131.243:5984/_cluster_setup -d '{"action": "enable_ cluster", "bind_address":"0.0.0.0", "username": "admin", "password":"admin", "port": 5984, "node_count": "3", "remote_node": "158.160.131.246", "remote_current_user": " admin", "remote_current_password": "admin" }'
{"ok":true}
curl -X POST -H "Content-Type: application/json" http://admin:admin@158.160.131.243:5984/_cluster_setup -d '{"action": "add_node", "host":"158.160.131.246", "port": 5984, "username": "admin", "password":"admin"}'
{"ok":true}

# Завершаем инициализацию кластера
curl -X POST -H "Content-Type: application/json" http://admin:admin@158.160.131.243:5984/_cluster_setup -d '{"action": "finish_ cluster"}'
{"ok":true}
curl http://admin:admin@158.160.131.243:5984:5984/_cluster_setup
{"state":"cluster_finished"}
curl http://admin:admin@158.160.131.243:5984/_membership
{"all_nodes":["couchdb@158.160.131.243","couchdb@158.160.131.241","couchdb@158.160.131.246"],"cluster_nodes":["couchdb@158.160.131.243","couchdb@158.160.131.241","couchdb@158.160.131.246"]}
```
> Кластер собран через web можно увидеть, что все базы данных реплецированы. 

> http://158.160.131.243:5984/_utils/#/_all_dbs 

> http://158.160.131.241:5984/_utils/#/_all_dbs 

> http://158.160.131.246:5984/_utils/#/_all_dbs 

>> При отключению любого сервера до 2 штук наши данные не пострадают. [CouchDB Cluster Theory](https://docs.couchdb.org/en/stable/cluster/theory.html)
