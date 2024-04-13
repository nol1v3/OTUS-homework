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

| Конфиг-сервера |
| ----------- |
| srv-ubu-mongodb-conf01    |
| srv-ubu-mongodb-conf02    |
| srv-ubu-mongodb-conf03    |

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
> Шаги необходимо выполнить для каждого сервера
[Установка сервера MongoDB (.deb)](#title1)





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
- Если сервер упал с ошибкой, необходимо посмотреть:
  1. **Лог-файл:** /var/log/mongodb/mongod.log.
  2. Проверить отступы в **конфиг-файле:** /etc/mongod.conf

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

