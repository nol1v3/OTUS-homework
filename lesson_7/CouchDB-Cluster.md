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
> Кластер собран, через web можно увидеть, что все базы данных реплецированы. 

> http://158.160.131.243:5984/_utils/#/_all_dbs 

> http://158.160.131.241:5984/_utils/#/_all_dbs 

> http://158.160.131.246:5984/_utils/#/_all_dbs 

>> При отключению любого сервера до 2 штук наши данные не пострадают. [CouchDB Cluster Theory](https://docs.couchdb.org/en/stable/cluster/theory.html)

## Проверяем кластер.
```bash
cat > /root/movies.db.dump.json << ENDJSON
{
  "docs": [
    {
      "_id": "_design/sample",
      "views": {
        "actors": {
          "map": "function(doc) { if (doc.actors) { for (i = 0; i < doc.actors.length; i++) { emit([doc.actors[i].first_name, doc.actors[i].last_name], doc.title); } } }"
        },
        "directors": {
          "map": "function(doc) { emit(doc.title, doc.director) }"
        },
        "actorsdirectors": {
          "map": "function(doc) { if (doc.actors) { for (i = 0; i < doc.actors.length; i++) { emit([doc.actors[i].first_name, doc.actors[i].last_name], [ doc.title , doc.director] ); } } }"
        },
        "genrecount": {
          "reduce": "_count",
          "map": "function (doc) { emit(doc.genre, doc.title) ; }"
        },
        "genre": {
          "map": "function (doc) { emit(doc.genre, doc.title) ; }"
        },
        "conflicts": {
          "map": "function (doc) {  if(doc._conflicts) {  emit(doc._conflicts, null); } }"
        }
      },
      "shows": {
        "title": "function(doc, req) { if (doc.title != null) { return '<h1>' + doc.title + '</h1>' } }",
        "detail": "function(doc, req) { var output ; if (doc.title !== null) { output =  '<h1>' + doc.title + '</h1>' ; output += '<p>' + doc.genre + '</p>' ; output += '<p>'  + doc.year    + '</p>' ; output += '<p>'  + doc.country + '</p>' ; output += '<p>'  + ' Director = ' + doc.director.last_name +  ' ' + doc.director.first_name + '</p>' ; output += '<h2>' + doc.summary + '</h2><ul>' ; for(i=0;i<doc.actors.length;i++) { output += '<li>' + doc.actors[i].first_name + ' ' + doc.actors[i].last_name + ' as ' + doc.actors[i].role + '</li>' ; } output += '</ul>' ; return output } }"
      },
      "language": "javascript"
    },
    {
      "_id": "f96b64a80ecaffd8c12dbd4e4f004b74",
      "title": "Spider-Man",
      "year": "2002",
      "genre": "Action",
      "summary": "On a school field trip, Peter Parker (Maguire) is bitten by a genetically modified spider. He wakes up the next morning with incredible powers. After witnessing the death of his uncle (Robertson), Parkers decides to put his new skills to use in order to rid the city of evil, but someone else has other plans. The Green Goblin (Dafoe) sees Spider-Man as a threat and must dispose of him.",
      "country": "USA",
      "director": {
        "last_name": "Raimi",
        "first_name": "Sam",
        "birth_date": "1959"
      },
      "actors": [
        {
          "first_name": "Tobey",
          "last_name": "Maguire",
          "birth_date": "1975",
          "role": "Spider-Man / Peter Parker"
        },
        {
          "first_name": "Kirsten",
          "last_name": "Dunst",
          "birth_date": "1982",
          "role": "Mary Jane Watson"
        },
        {
          "first_name": "Willem",
          "last_name": "Dafoe",
          "birth_date": "1955",
          "role": "Green Goblin / Norman Osborn"
        }
      ]
    },
    {
      "_id": "f96b64a80ecaffd8c12dbd4e4f0088d0",
      "title": "Unforgiven",
      "year": "1992",
      "genre": "Western",
      "summary": "The town of Big Whisky is full of normal people trying to lead quiet lives. Cowboys try to make a living. Sheriff 'Little Bill' tries to build a house and keep a heavy-handed order. The town whores just try to get by.Then a couple of cowboys cut up a whore. Unsatisfied with Bill's justice, the prostitutes put a bounty on the cowboys. The bounty attracts a young gun billing himself as 'The Schofield Kid', and aging killer William Munny. Munny reformed for his young wife, and has been raising crops and two children in peace. But his wife is gone. Farm life is hard. And Munny is no good at it. So he calls his old partner Ned, saddles his ornery nag, and rides off to kill one more time, blurring the lines between heroism and villainy, man and myth.",
      "country": "USA",
      "director": {
        "last_name": "Eastwood",
        "first_name": "Clint",
        "birth_date": "1930"
      },
      "actors": [
        {
          "first_name": "Clint",
          "last_name": "Eastwood",
          "birth_date": "1930",
          "role": "William Munny"
        },
        {
          "first_name": "Gene",
          "last_name": "Hackman",
          "birth_date": "1930",
          "role": "Little Bill Dagget"
        },
        {
          "first_name": "Morgan",
          "last_name": "Freeman",
          "birth_date": "1937",
          "role": "Ned Logan"
        }
      ]
    },
    {
      "_id": "f96b64a80ecaffd8c12dbd4e4f00913b",
      "title": "A History of Violence",
      "year": "2005",
      "genre": "Crime",
      "summary": "Tom Stall, a humble family man and owner of a popular neighborhood restaurant, lives a quiet but fulfilling existence in the Midwest. One night Tom foils a crime at his place of business and, to his chagrin, is plastered all over the news for his heroics. Following this, mysterious people follow the Stalls' every move, concerning Tom more than anyone else. As this situation is confronted, more lurks out over where all these occurrences have stemmed from compromising his marriage, family relationship and the main characters' former relations in the process.",
      "country": "USA",
      "director": {
        "last_name": "Cronenberg",
        "first_name": "David",
        "birth_date": "1943"
      },
      "actors": [
        {
          "first_name": "Ed",
          "last_name": "Harris",
          "birth_date": "1950",
          "role": "Carl Fogarty"
        },
        {
          "first_name": "Vigo",
          "last_name": "Mortensen",
          "birth_date": "1958",
          "role": "Tom Stall"
        },
        {
          "first_name": "Maria",
          "last_name": "Bello",
          "birth_date": "1967",
          "role": "Eddie Stall"
        },
        {
          "first_name": "William",
          "last_name": "Hurt",
          "birth_date": "1950",
          "role": "Richie Cusack"
        }
      ]
    },
    {
      "_id": "f96b64a80ecaffd8c12dbd4e4f0097ad",
      "title": "Marie Antoinette",
      "year": "2006",
      "genre": "Drama",
      "summary": "Based on Antonia Fraser's book about the ill-fated Archduchess of Austria and later Queen of France, 'Marie Antoinette' tells the story of the most misunderstood and abused woman in history, from her birth in Imperial Austria to her later life in France.",
      "country": "USA",
      "director": {
        "last_name": "Coppola",
        "first_name": "Sofia",
        "birth_date": "1971"
      },
      "actors": [
        {
          "first_name": "Kirsten",
          "last_name": "Dunst",
          "birth_date": "1982",
          "role": "Marie Antoinette"
        },
        {
          "first_name": "Jason",
          "last_name": "Schwartzman",
          "birth_date": "1980",
          "role": "Louis XVI"
        }
      ]
    },
    {
      "_id": "f96b64a80ecaffd8c12dbd4e4f00bf3f",
      "title": "The Social network",
      "year": "2010",
      "genre": "Drama",
      "summary": "On a fall night in 2003, Harvard undergrad and computer programming genius Mark Zuckerberg sits down at his computer and heatedly begins working on a new idea. In a fury of blogging and programming, what begins in his dorm room soon becomes a global social network and a revolution in communication. A mere six years and 500 million     friends later, Mark Zuckerberg is the youngest billionaire in history... but for this entrepreneur, success leads to both personal and legal complications.",
      "country": "USA",
      "director": {
        "last_name": "Fincher",
        "first_name": "David",
        "birth_date": "1962"
      },
      "actors": [
        {
          "first_name": "Jesse",
          "last_name": "Eisenberg",
          "birth_date": "1983",
          "role": "Mark Zuckerberg"
        },
        {
          "first_name": "Rooney",
          "last_name": "Mara",
          "birth_date": "1985",
          "role": "Erica Albright"
        },
        {
          "first_name": "Andrew",
          "last_name": "Garfield",
          "birth_date": "1983",
          "role": "Eduardo Saverin"
        },
        {
          "first_name": "Justin",
          "last_name": "Timberlake",
          "birth_date": "1981",
          "role": "Sean Parker"
        }
      ]
    }
  ]
}
ENDJSON

curl -X PUT http://admin:admin@localhost:5984/movies_restored
{"ok":true}
curl -d @/root/movies.db.dump.json -H "Content-type: application/json" -X POST http://admin:dba@localhost:5984/movies_restored/_bulk_docs
[{"ok":true,"id":"_design/sample","rev":"1-4b5394ab93ce1a821ec740a0ec950106"},{"ok":true,"id":"f96b64a80ecaffd8c12dbd4e4f004b74","rev":"1-55f2ec9c7c53fec192cdb16c9c4f50 c9"},{"ok":true,"id":"f96b64a80ecaffd8c12dbd4e4f0088d0","rev":"1-576d70babcd04fed2918f5c543bb7cf6"},{"ok":true,"id":"f96b64a80ecaffd8c12dbd4e4f00913b","rev":"1-cba8edd3 907e29925aac6ffdbcf921a5"},{"ok":true,"id":"f96b64a80ecaffd8c12dbd4e4f0097ad","rev":"1-b9481b94d092b1a65b64589492d2b49e"},{"ok":true,"id":"f96b64a80ecaffd8c12dbd4e4f00b f3f","rev":"1-d3ea1b5a4e6b014c10123f4e9f5fbad8"}]
```

> Через web можно увидеть, что все база данных movies_restored реплецирована.
>>Следовательно потеря сервера нам не угрожает, так как данные остануться на двух других.

> http://158.160.131.243:5984/_utils/#/_all_dbs 

> http://158.160.131.241:5984/_utils/#/_all_dbs 

> http://158.160.131.246:5984/_utils/#/_all_dbs
