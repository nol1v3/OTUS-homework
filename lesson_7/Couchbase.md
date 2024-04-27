# Настройка кластера Couchbase.
> Установка выполнена через **сервис:** YandexCloud.
>> Установка будет выполнена из *.deb пакетов по **руководству:** [Couchbase Ubuntu](https://docs.couchbase.com/server/current/install/install-intro.html)

| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-couchbase01    | 158.160.135.152   |
| srv-ubu-couchbase02    | 158.160.159.186   |

## Установка Couchbase.
```bash
curl -O https://packages.couchbase.com/releases/couchbase-release/couchbase-release-1.0-noarch.deb
sudo dpkg -i ./couchbase-release-1.0-noarch.deb
sudo apt-get update
sudo apt-get install couchbase-server-community
```

## Создаем кластер Couchbase.
```bash
couchbase-cli cluster-init -c 158.160.135.152 \
--cluster-username Administrator \
--cluster-password Administrator \
--services data,index,query \
--cluster-ramsize 512 \
--cluster-index-ramsize 256

SUCCESS: Cluster initialized
```
> Либо используем web-интерфейс на адресе: 158.160.135.152:8091
>> Выполняем шаги: [Couchbase create cluster](https://docs.couchbase.com/server/current/manage/manage-nodes/create-cluster.html)

## Добавляем ноду в кластер Couchbase.
```bash
couchbase-cli server-add -c 158.160.135.152:8091 \
--username Administrator \
--password password \
--server-add https://158.160.159.186:8091 \
--server-add-username Administrator \
--server-add-password Administrator \
--services data,index,query

SUCCESS: Server added
```

## Проводим ребалансировку кластера Couchbase.
```bash
couchbase-cli rebalance -c 158.160.135.152:8091 \
--username Administrator \
--password Administrator

SUCCESS: Rebalance complete
```
> Либо используем web-интерфейс на адресе: 158.160.135.152:8091
>> Выполняем шаги: [Couchbase Add and rebalance node](https://docs.couchbase.com/server/current/manage/manage-nodes/add-node-and-rebalance.html)
>>> Так же присоединение сервера можно проводить через web-интерфейс: 158.160.159.186:8091
>>>> Необходимо выполнить шаги: [Couchbase Join and rebalance node](https://docs.couchbase.com/server/current/manage/manage-nodes/add-node-and-rebalance.html)

## Проверяем корректность сборки кластера Couchbase.
```bash
couchbase-cli server-list -c 158.160.135.152:8091 \
--username Administrator \
--password Administrator

ns_1@158.160.135.152 158.160.135.152:8091 healthy active
ns_1@158.160.159.186 158.160.159.186:8091 healthy active
```
> Либо используем web-интерфейс на адресе: 158.160.135.152:8091
>> Выполняем шаги: [Couchbase List node](https://docs.couchbase.com/server/current/manage/manage-nodes/list-cluster-nodes.html)
>>> Теперь при потере одного узла наши данные будут в безопастности.

## Automatic Failover Couchbase.
> Автоматическое переключение сервера на котором произошел сбой настроено на интервал в 120 сек.
>> Automatic Failover: [Couchbase Failover](https://docs.couchbase.com/server/current/learn/clusters-and-availability/failover.html)
```
Couchbase-cli Setting-AutoFailover -c 10.142.181.101 --username Administrator \
 --password Administrator --enable-auto-failover 1 --auto-failover-timeout 30
```
> Данной командой можно изменить интервал времени.

## Тест
> Загружаем данные из sample-bucket: beer-sample
>> Отключаем один из серверов:
```bash
systemctl stop couchbase-server.service
```
>> Выполняем failover через web-интерфейс на адресе: 158.160.135.152:8091
```bash
couchbase-cli failover -c 158.160.135.152:8091 \
--username Administrator \
--password Administrator \
--server-failover 158.160.159.186:8091 --hard

SUCCESS: Server failed over
```
>> Hard Failover: [Couchbase Hard Failover](https://docs.couchbase.com/server/current/manage/manage-nodes/failover-hard.html)
>>> Данные полностью сохранены на bucket'ах сервера: 158.160.135.152
>>>> Для полного удаления узла и переход на один узел необходимо выполнить ребалансировку.
