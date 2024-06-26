# Установка и настройка кластера Redis.
> Установка выполнена через **сервис:** YandexCloud.

>> Установка будет выполнена из *.deb пакетов по **руководству:** [Redis Ubuntu](https://redis.io/docs/latest/operate/oss_and_stack/install/install-stack/linux/)
>>> Установка кластера будет выполнена по **руководству**: [Redis Cluster](https://redis.io/docs/latest/operate/oss_and_stack/management/scaling/)

| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-redis01    | 158.160.131.260   |
| srv-ubu-redis02    | 158.160.131.261   |
| srv-ubu-redis03    | 158.160.131.262   |
| srv-ubu-redis04    | 158.160.131.273   |
| srv-ubu-redis05    | 158.160.131.274   |
| srv-ubu-redis06    | 158.160.131.275   |

## Установка сервера Redis. (На всех серверах)
```bash
curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
sudo chmod 644 /usr/share/keyrings/redis-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
sudo apt-get update
sudo apt-get install redis-stack-server
```

## Настройка кластера Redis.
```bash
systemctl enable redis.service
systemctl stop redis.service
vi /etc/redis.conf

bind 158.160.131.260 #Указать свой IP-адрес на каждом сервере
protected-mode no
port 6379
cluster-enabled yes
cluster-config-file nodes.conf
cluster-node-timeout 15000
appendonly yes

systemctl start redis.service
```

> Сервер должен запуститься без ошибок, если сервер упал необходимо посмотреть **лог-файл:** /var/log/redis/redis.log
>> Собирем кластер redis

```bash
redis-cli --cluster create 158.160.131.260:6379 158.160.131.261:6379 158.160.131.262:6379 158.160.131.273:6379 158.160.131.274:6379 158.160.131.275:6379 --cluster-replicas 1

>>> Performing hash slots allocation on 6 nodes...

Master[0] -> Slots 0 - 5460

Master[1] -> Slots 5461 - 10922

Master[2] -> Slots 10923 - 16383

Adding replica 158.160.131.273:6379 to 158.160.131.260:6379

Adding replica 158.160.131.274:6379 to 158.160.131.261:6379

Adding replica 158.160.131.275:6379 to 158.160.131.262:6379

M: 4394d8eb03de1f524b56cb385f0eb9052ce65283 158.160.131.260:6379

   slots:[0-5460] (5461 slots) master

M: 5cc0f693985913c553c6901e102ea3cb8d6678bd 158.160.131.261:6379

   slots:[5461-10922] (5462 slots) master

M: 22de56650b3714c1c42fc0d120f80c66c24d8795 158.160.131.262:6379

   slots:[10923-16383] (5461 slots) master

S: 8675cd30fdd4efa088634e50fbd5c0675238a35e 158.160.131.275:6379

   replicates 22de56650b3714c1c42fc0d120f80c66c24d8795

S: ad0f5210dda1736a1b5467cd6e797f011a192097 158.160.131.273:6379

   replicates 4394d8eb03de1f524b56cb385f0eb9052ce65283

S: 184ada329264e994781412f3986c425a248f386e 158.160.131.274:6379

   replicates 5cc0f693985913c553c6901e102ea3cb8d6678bd

Can I set the above configuration? (type 'yes' to accept):

yes

>>> Nodes configuration updated

>>> Assign a different config epoch to each node

>>> Sending CLUSTER MEET messages to join the cluster

Waiting for the cluster to join

.

>>> Performing Cluster Check (using node 158.160.131.260)

M: 4394d8eb03de1f524b56cb385f0eb9052ce65283 158.160.131.260

   slots:[0-5460] (5461 slots) master

   1 additional replica(s)

S: 184ada329264e994781412f3986c425a248f386e 158.160.131.273:6379

   slots: (0 slots) slave

   replicates 5cc0f693985913c553c6901e102ea3cb8d6678bd

M: 5cc0f693985913c553c6901e102ea3cb8d6678bd 158.160.131.261:6379

   slots:[5461-10922] (5462 slots) master

   1 additional replica(s)

M: 22de56650b3714c1c42fc0d120f80c66c24d8795 158.160.131.262:6379

   slots:[10923-16383] (5461 slots) master

   1 additional replica(s)

S: ad0f5210dda1736a1b5467cd6e797f011a192097 158.160.131.274:6379

   slots: (0 slots) slave

   replicates 4394d8eb03de1f524b56cb385f0eb9052ce65283

S: 8675cd30fdd4efa088634e50fbd5c0675238a35e 158.160.131.275:6379

   slots: (0 slots) slave

   replicates 22de56650b3714c1c42fc0d120f80c66c24d8795

[OK] All nodes agree about slots configuration.

>>> Check for open slots...

>>> Check slots coverage...

[OK] All 16384 slots covered.

redis-cli -h 158.160.131.260 -p 6379 cluster nodes
```

> Три первых узла будут главными, а остальные — подчиненными.
> В Redis - нет полной поддержки JSON-файлов, воспользуемся PYTHON. Размер JSON ~50мб.

```python
import json
import redis
r = redis.StrictRedis(host='localhost', port=6379, db=1)
with open('json_test.json') as data_file:
    test_data = json.load(data_file)
r.set('test_json', test_data)
```

>Сохранение json во всех структурах данных заняло примерно 5.5с (с учетом нагрузки на сеть). Получение JSON заняло 1.1 секунду.

>Для хранения JSON можно так же можно добавить модуль для нового типа данных JSON: [RedisJSON](https://github.com/RedisJSON/RedisJSON)

>Установка RedisJSON: [RedisJSON install](https://gist.github.com/lmj0011/820eea392f6f43c755fadc2ba56b69e9)
