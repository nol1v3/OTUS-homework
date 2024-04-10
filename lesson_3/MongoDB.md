# Первичная настройка и установка сервера MongoDB (.deb)
Установка будет выполнена из *.deb пакетов по **руководству:** [MongoDB Ubuntu](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu/)

## Первичная настройка параметров ОС для MongoDB.
- Согласно руководству необходимо выполнить выключение **больших страниц ОС:** [MongoDB THP](https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/)
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never

sudo vim /etc/default/grub

# Добавляем строку

GRUB_CMDLINE_LINUX="transparent_hugepage=never"

sudo grub-mkconfig -o /boot/grub/grub.cfg
```
- Согласно руководству необходимо выставить **лимиты ОС:** [MongoDB ulimit Settings](https://www.mongodb.com/docs/manual/reference/ulimit/)
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
- Для применения параметров необходимо выполнить **перезагрузку системы**.

## Установка сервера MongoDB (.deb)
- Определяем версию сервера.
```bash
cat /etc/lsb-release
```
- Устанавливаем зависимости для добавления репозитория.
```bash
sudo apt-get install gnupg curl
```
- Добавляем ключ к репозиторию и репозиторий.
```
sudo curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | \
   sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg \
   --dearmor

sudo echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list
```
- Обновляем список пакетов и устанавливаем сервер MongoDB.
```bash
sudo apt-get update
sudo apt-get install -y mongodb-org
```
- Запускаем сервер MongoDB.
```bash
sudo systemctl start mongod
sudo systemctl status mongod
```

## Включение авторизации MongoDB.
- Подключаемся к серверу MongoDB.
```bash
sudo mongosh
```
- Создаем пользователя.
```
use admin
show roles
show users
db.createUser({ user: "root_mongodb", pwd: passwordPrompt(), roles: [ "root" ] })
show users
```
- Включаем авторизацию на сервере MongoDB.
```bash
sudo vim /etc/mongod.conf

# Раскомментировать секцию security и установить параметры
security.authorization: enabled

sudo systemctl restart mongod
sudo systemctl status mongod
```
- Если сервер запустился без ошибок, то все сделано правильно.  
- Если сервер упал с ошибкой, необходимо посмотреть **лог-файл**:/var/log/mongodb/mongod.log и проверить отступы в **конфиг-файле**:/etc/mongod.conf

## Авторизация на сервере MongoDB.
```bash
sudo mongosh
```
```
use admin
show users
MongoServerError[Unauthorized]: Command usersInfo requires authentication
```
- Из сообщения видно, что данная команда доступна только авторизованным пользователем.
```
db.auth( "root_mongodb", "root_mongodb" )
{ ok: 1 } # Авторизация прошла успешно
show users
```

## Заполним базу MongoDB данными.
```
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
```
- Проверяем данные.
```
db.purchases.find().limit(10)
```
- Выполним обновление данных.
```
db.purchases.find({transaction_number: '#0'})
[
  {
    _id: ObjectId('661706d193504b8010ef634b'),
    transaction_number: '#0',
    transaction: 'tx_0',
    transaction_price: 39,
    transaction_type: 'credit card',
    store_name: 'makro'
  }
]

db.purchases.updateOne({transaction_number: '#0'}, {$set: {transaction_price:41}})
{
  acknowledged: true,
  insertedId: null,
  matchedCount: 1,
  modifiedCount: 1,
  upsertedCount: 0
}

db.purchases.find({transaction_number: '#0'})
[
  {
    _id: ObjectId('661706d193504b8010ef634b'),
    transaction_number: '#0',
    transaction: 'tx_0',
    transaction_price: 41,
    transaction_type: 'credit card',
    store_name: 'makro'
  }
]
```
- Выполним вставку данных.
```
db.purchases.find({transaction_number: '#100001'})

db.purchases.insertOne({transaction_number: '#100001', transaction: 'tx_100001', transaction_price: 10000, transaction_type: 'credit card', store_name: 'cna'});
{
  acknowledged: true,
  insertedId: ObjectId('66170b66da8a9eadfdef634b')
}
db.purchases.find({transaction_number: '#100001'})

[
  {
    _id: ObjectId('66170b66da8a9eadfdef634b'),
    transaction_number: '#100001',
    transaction: 'tx_100001',
    transaction_price: 10000,
    transaction_type: 'credit card',
    store_name: 'cna'
  }
]
```
- Создание индекса.
```bash
cat /home/malyushin/1.js
use test_transactions
db.purchases.find({store_name: 'edgards'}).count()
time mongosh -u root_mongodb -p root_mongodb --authenticationDatabase admin --quiet < 1.js
real    0m1.697s
user    0m1.760s
sys     0m0.085s

mongosh -u root_mongodb -p root_mongodb --authenticationDatabase admin
use test_transactions
# Создаем индекс с названием myIndex и прямой последовательностью
db.purchases.createIndex({store_name: 1}, {name: "myIndex"});
# Проверяем что индекс создался
db.purchases.getIndexes()

time mongosh -u root_mongodb -p root_mongodb --authenticationDatabase admin --quiet < 1.j
real    0m0.653s
user    0m0.685s
sys     0m0.098s
```
- Верное индексирование повышает производительность базы данных.
