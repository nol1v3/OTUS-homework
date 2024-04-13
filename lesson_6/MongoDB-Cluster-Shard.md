# Настройка кластера MongoDB и Шардирование.
> Установка выполнена через **сервис:** YandexCloud.
>> Установка будет выполнена из *.deb пакетов по **руководству:** [MongoDB Ubuntu](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)
>>> Установка кластера будет выполнена по **руководству**: [MongoDB Replica Set](https://www.mongodb.com/docs/manual/replication/)  
>>>> Установка шардирования выполнена по **руководству**: [MongoDB Sharding](https://www.mongodb.com/docs/manual/sharding/)  

## Настройка конфиг-серверов MongoDB.
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

## Подготовка кластера.
> Для инициализации кластера с авторизацией необходимо выполнить шаги **по руководству**: [MongoDB Security Replica set](https://www.mongodb.com/docs/manual/tutorial/enable-authentication/#std-label-enable-access-control)

```bash
sudo vim /etc/mongod.conf

# Раскомментировать секцию security и установить параметр
security.keyFile: /var/lib/mongo/repsetkey/keyfile

# Останавливаем сервис MongoDB 
sudo systemctl stop mongod.service
# Создаем папку для ключа шифрования
sudo mkdir -p /var/lib/mongo/repsetkey
# Меняем права на папку
sudo chown mongod:mongod -R /var/lib/mongo/repsetkey/keyfile
```

> Создаем и копируем repsetkey между серверами MongoDB.

```bash
# Создаем ключ шифрования
sudo openssl rand -base64 756 > /var/lib/mongo/repsetkey/keyfile
# Меняем права и влладельца на ключ
sudo chmod 400 /var/lib/mongo/repsetkey/keyfile
sudo chown mongod:mongod -R /var/lib/mongo/repsetkey/keyfile
# Передаем ключ шифрования на сервера
sudo rsync -av /var/lib/mongo/repsetkey/keyfile root@{{_SERVER_NAME_}}:/var/lib/mongo/repsetkey/keyfile
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

db.auth( "root_mongodb", "root_mongodb" )
{ ok: 1 }

db.serverCmdLineOpts()

# Создаем конфигурацию Replica set для конфиг серверов
use admin
rsconf = {
... _id: "rs0",
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
>> Изменения требуют параметры на данные из таблицы.
```
_id: "rs0"
_id: 0, host: "srv-ubu-mongodb-conf01:27017"
_id: 1, host: "srv-ubu-mongodb-conf02:27017"
_id: 2, host: "srv-ubu-mongodb-conf03:27017"
```
[Установка сервера MongoDB (.deb)](#title1)
