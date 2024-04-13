# Настройка кластера MongoDB и Шардирование.
> Установка выполнена через **сервис:** YandexCloud.
>> Установка будет выполнена из *.deb пакетов по **руководству:** [MongoDB Ubuntu](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)
>>> Установка кластера будет выполнена по **руководству**: [MongoDB Replica Set](https://www.mongodb.com/docs/manual/replication/)  
>>>> Установка шардирования выполнена по **руководству**: [MongoDB Sharding](https://www.mongodb.com/docs/manual/sharding/)  

## Настройка серверов MongoDB.
> Первичная настройка параметров ОС для MongoDB.   
>> Согласно руководству необходимо выполнить выключение **больших страниц ОС:** [MongoDB THP](https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/)
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never

sudo vim /etc/default/grub

# Добавляем строку

GRUB_CMDLINE_LINUX="transparent_hugepage=never"

sudo grub-mkconfig -o /boot/grub/grub.cfg
```
> Согласно руководству необходимо выставить **лимиты ОС:** [MongoDB ulimit Settings](https://www.mongodb.com/docs/manual/reference/ulimit/)
```bash
sudo vim /etc/security/limits.conf

# Добавляем строки
  
# MongoDB limits
mongod soft nofile 64000
mongod hard nofile 64000
mongod soft nproc 64000 
mongod hard nproc 64000
mongod soft fsize unlimited
mongod hard fsize unlimited
mongod soft cpu unlimited
mongod hard cpu unlimited
mongod soft as unlimited
mongod hard as unlimited
mongod soft memlock unlimited
mongod hard memlock unlimited
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

## <a id="title1">Установка сервера MongoDB (.deb)</a>

| Конфиг-сервера (rs0) | Дата-сервера (rs1) | Дата-сервера (rs2) |
| ----------- | ----------- | ----------- |
| srv-ubu-mongodb-conf01    | srv-ubu-mongodb-data01    | srv-ubu-mongodb-data04    |
| srv-ubu-mongodb-conf02    | srv-ubu-mongodb-data02    | srv-ubu-mongodb-data05    |
| srv-ubu-mongodb-conf03    | srv-ubu-mongodb-data03    | srv-ubu-mongodb-data06    |

> Определяем версию сервера.
```bash
cat /etc/lsb-release
```
> Устанавливаем зависимости для добавления репозитория.
```bash
sudo apt-get install gnupg curl
```
> Добавляем ключ к репозиторию и репозиторий.
```
sudo curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

sudo echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```
> Обновляем список пакетов и устанавливаем сервер MongoDB.
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```
> Запускаем сервер MongoDB.
```bash
sudo systemctl start mongod
sudo systemctl status mongod
```
> Шаги необходимо выполнить для **каждого сервера из списка:**  
[Установка сервера MongoDB (.deb)](#title1)

## Включение авторизации MongoDB.
> Подключаемся к серверу MongoDB.
```bash
sudo mongosh
```
> Создаем пользователя. Пользователя можно создать на одном сервере, т.к он будет реплицирован.
```
use admin
show roles
show users
db.createUser({ user: "root_mongodb", pwd: passwordPrompt(), roles: [ "root" ] })
show users
```
> Включаем авторизацию на сервере MongoDB.
```bash
sudo vim /etc/mongod.conf

# Раскомментировать секцию security и установить параметры
security.authorization: enabled

# Отвязываем сервер от localhost
net.bindIpAll: true

# Каждая база в отдельной директории
storage.directoryPerDB: true

sudo systemctl restart mongod
sudo systemctl status mongod
```
> Если сервер запустился без ошибок, то все сделано правильно.


> Если сервер упал с ошибкой, необходимо посмотреть:
>> Проверить отступы в **конфиг-файле:** /etc/mongod.conf  
>> Проверить наличие ошибок в **лог-файл:** /var/log/mongodb/mongod.log 


> Шаги необходимо выполнить для **каждого сервера из списка:**  
[Установка сервера MongoDB (.deb)](#title1)

## Подготовка кластера MongoDB
> Для инициализации кластера с авторизацией необходимо выполнить шаги **по руководству**: [MongoDB Security Replica set](https://www.mongodb.com/docs/manual/tutorial/enable-authentication/#std-label-enable-access-control)

```bash
sudo vim /etc/mongod.conf

# Раскомментировать секцию security и установить параметр
security.keyFile: /var/lib/repsetkey/keyfile

# Останавливаем сервис MongoDB 
sudo systemctl stop mongod.service
# Создаем папку для ключа шифрования
sudo mkdir -p /var/lib/repsetkey
# Меняем права на папку
sudo chown mongodb:mongodb -R /var/lib/repsetkey
```

> Создаем и копируем repsetkey между серверами MongoDB (общий ключ для всех Replica Set).

```bash
# Создаем ключ шифрования
sudo openssl rand -base64 756 > /var/lib/repsetkey/keyfile
# Меняем права и влладельца на ключ
sudo chmod 400 /var/lib/repsetkey/keyfile
sudo chown mongodb:mongodb -R /var/lib/repsetkey/keyfile
# Передаем ключ шифрования на сервера
sudo rsync -av /var/lib/repsetkey/keyfile root@{{_SERVER_NAME_on_rs{0..2}}}:/var/lib/repsetkey/keyfile
# Запускаем MongoDB
sudo systemctl start mongod
```
> Если сервер запустился без ошибок, то все сделано правильно.


> Если сервер упал с ошибкой, необходимо посмотреть:
>> Проверить отступы в **конфиг-файле:** /etc/mongod.conf  
>> Проверить наличие ошибок в **лог-файл:** /var/log/mongodb/mongod.log  

> Шаги необходимо выполнить для **каждого сервера из списка:**  
[Установка сервера MongoDB (.deb)](#title1)

## Инициализация кластера MongoDB

> Инициализацию кластера необходимо проводить на отдельном сервере.

```bash
# Подключаемся к MongoDB и проверям параметры запуска
sudo mongosh

db.auth( "root_mongodb", "root_mongodb" )
{ ok: 1 }

db.serverCmdLineOpts()

# Меняем конфигурацию сервера MongoDB 
sudo vim /etc/mongod.conf

# Раскомментируем секцию replication и устанавливаем параметры
replication.replSetName: rs0
replication.oplogSizeMB: 500

# Перезагружаем сервер MongoDB 
sudo systemctl restart mongod.service 

# Подключаемся к MongoDB и проверям параметры запуска
sudo mongosh

# Создаем конфигурацию Replica set для конфиг серверов
use admin
rsconf = {
... _id: "rs0",
... configsvr: true,
... members: [
... { _id: 0, host: "srv-ubu-mongodb-conf01:27017" },
... { _id: 1, host: "srv-ubu-mongodb-conf02:27017" },
... { _id: 2, host: "srv-ubu-mongodb-conf03:27017" }
... ]
... }

# Проверяем конфигурацию ReplicaSet
printjson(rsconf)

# Инициализируем кластер MongoDB 
rs.initiate(rsconf)
{ "ok" : 1 }

# В консоли должно поменяться приглашение ввода
# На PRIMARY
rs0:PRIMARY>
# На SECONDARY
rs0:SECONDARY>

# Полная статистика статистика о Replica Set
rs.config()
```

> Шаги необходимо выполнить для **каждого сервера из списка:**  
[Установка сервера MongoDB (.deb)](#title1)
>> Изменения требуют параметры. (из таблицы)
```
_id: "rs0 - rs2"
_id: 0, host: "srv-ubu-mongodb-conf/data{01..06}:27017"
_id: 1, host: "srv-ubu-mongodb-conf/data{01..06}:27017"
_id: 2, host: "srv-ubu-mongodb-conf/data{01..06}:27017"
```
Плюсы Replica set:
- при отказе мастер-сервера, происходит автоматический выбор нового мастер-сервера. (потеря данных не происходит)
- при отказе мастер-сервера, происходит автоматический выбор нового мастер-сервера. (минимальный простой для приложения)
- возможность организации read concern/write concern. (возможность балансировки нагрузки (на уровне подключения))

Минусы Replica set:
- Сложность в администрировании кластера.

## Настройка Шардирования configsvr MongoDB 
> Настройка конфиг-серверов MongoDB.
>> Остановка серверов происходит по схеме: SECONDARY -> SECONDARY -> PRIMARY  
>> Запуск происходит в обратном порядке: PRIMARY -> SECONDARY -> SECONDARY  
>> При данной остановке и запуске смены мастера не будет.
```bash
sudo systemctl stop mongod.service
vim /etc/mongod.conf

# Раскомментируем секцию sharding и устанавливаем параметры
sharding.clusterRole: "configsvr"

sudo systemctl start mongod.service
```
> Если сервер запустился без ошибок, то все сделано правильно.


> Если сервер упал с ошибкой, необходимо посмотреть:
>> Проверить отступы в **конфиг-файле:** /etc/mongod.conf  
>> Проверить наличие ошибок в **лог-файл:** /var/log/mongodb/mongod.log  

> Шаги необходимо выполнить для **каждого сервера из списка Конфиг-сервера (rs0):**  
[Установка сервера MongoDB (.deb)](#title1)

## Настройка Шардирования shardsvr MongoDB 
> Настройка дата-серверов MongoDB.
>> Остановка серверов происходит по схеме: SECONDARY -> SECONDARY -> PRIMARY  
>> Запуск происходит в обратном порядке: PRIMARY -> SECONDARY -> SECONDARY  
>> При данной остановке и запуске смены мастера не будет.

```bash
sudo systemctl stop mongod.service
vim /etc/mongod.conf

# Раскомментируем секцию sharding и устанавливаем параметры
sharding.clusterRole: "shardsvr"

sudo systemctl start mongod.service
```
> Если сервер запустился без ошибок, то все сделано правильно.


> Если сервер упал с ошибкой, необходимо посмотреть:
>> Проверить отступы в **конфиг-файле:** /etc/mongod.conf  
>> Проверить наличие ошибок в **лог-файл:** /var/log/mongodb/mongod.log

> Шаги необходимо выполнить для **каждого сервера из списка Конфиг-сервера (rs1/rs2):**  
[Установка сервера MongoDB (.deb)](#title1)

## Настройка балансировщика mongos

| сервер-балансировщик | 
| ----------- | 
| srv-ubu-mongodb-mongos01 |

> Определяем версию сервера.
```bash
cat /etc/lsb-release
```
> Устанавливаем зависимости для добавления репозитория.
```bash
sudo apt-get install gnupg curl
```
> Добавляем ключ к репозиторию и репозиторий.
```
sudo curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

sudo echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```
> Обновляем список пакетов и устанавливаем сервер MongoDB.
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```

> Редактируем конфигурационный файл mongos.
```bash
vim /etc/mongod.conf

# Раскомментируем секцию sharding и устанавливаем параметры
sharding.configDB: "srv-ubu-mongodb-conf01:27017,srv-ubu-mongodb-conf02:27017,srv-ubu-mongodb-conf03:27017"

# Отвязываем сервер от localhost
net.bindIpAll: true

Раскомментировать секцию security и установить параметры
security.keyFile: /var/lib/repsetkey/keyfile

sudo systemctl start mongos.service
```

## Добавление дата-серверов в шардирование.

> Подключаемся к сервер-балансировщику mongos.
```bash
mongosh --host srv-ubu-mongodb-mongos01 --port 27017
sh.addShard("rs1/srv-ubu-mongodb-data01:27017,srv-ubu-mongodb-data02:27017,srv-ubu-mongodb-data03:27017")
sh.addShard("rs2/srv-ubu-mongodb-data04:27017,srv-ubu-mongodb-data05:27017,srv-ubu-mongodb-data06:27017")
sh.status()

use admin
sh.enableSharding("test_transactions")
sh.status()
```

## Проверяем распределение данных по шардам.
> Ключом шардирование выбираем поле transaction_price, так как оно может имеет множество значений.
>> Так же создадим хешированный индекс по полю transaction_price, что позволит нам избежать многочисленных записей в последний чанк.
```
use test_transactions
db.purchases.createIndex({"transaction_price": "hashed"});

use admin
db.runCommand({shardCollection: "test_transactions.purchases", key: {"transaction_price": "hashed"}})

use test_transactions
var transactions = []

for (var x = 0; x < 100000 ; x++) {
 var transaction_types = ["credit card", "cash", "account"];
 var store_names = ["edgards", "cna", "makro", "picknpay", "checkers"];
 var random_transaction_type = Math.floor(Math.random() * (2 - 0 + 1)) + 0;
 var random_store_name = Math.floor(Math.random() * (4 - 0 + 1)) + 0;
 transactions.push({
   transaction_number: '#' + x,
   transaction: 'tx_' + x,
   transaction_price: Math.round(Math.random()*1000),
   transaction_type: transaction_types[random_transaction_type],
   store_name: store_names[random_storename]
   });
 }

db.purchases.insertMany(transactions)

sh.status()
```
- Сначала чанки сохраняются на primary шард, а затем мигрируют на второй шард, пока количество не выровняется, причем их диапазоны при этом тоже могут изменяться.

Плюсы MongoDB Sharding:
- при отказе мастер-сервера конфиг-сервера или дата-сервера, происходит автоматический выбор нового мастер-сервера. (потеря данных не происходит)
- при отказе мастер-сервера конфиг-сервера или дата-сервера, происходит автоматический выбор нового мастер-сервера. (минимальный простой для приложения)
- автоматический балансировка нагрузки на основе ключа-шардирования.
- уменьшение накладных расходов для подключений из-за mongos

Минусы MongoDB Sharding:
- Большая сложность в администрировании кластера.
- Взаимная работа администраторов и разработчиков для выбора оптимальных решений шардирования.
- "Ограничение" по предикатам запросов, обязательно в предикат должен входит ключ шардирования.
- Избыточность серверов.
- Возможность появления горячих чанков на запись/чтение.

## Отказоустойчивость Replica set
- При настройке кластеризации и шардирования состоящих из 3 серверов, отказаустойчивость поддерживается до 1 сервера. Для большей отказоустойчивости необходимы дополнительные мощности.

| Количество серверов | Отказоустойчивость | Для избрания новых первичных выборов необходимо большинство |
| ----------- | ----------- | ----------- |
| 3   | 1   | 2    |
| 4    | 1    | 3   |
| 5    | 2   | 3   |
