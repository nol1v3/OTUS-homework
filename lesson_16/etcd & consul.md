# Установка и настройка кластера Etcd.
> Установка выполнена через **сервис:** YandexCloud.

>> Установка будет выполнена из *.deb пакетов по **руководству:** [Etcd Install](https://etcd.io/docs/v3.5/install/)
>>> Установка кластера будет выполнена по **руководству**: [Etcd Cluster](https://etcd.io/docs/v3.5/tutorials/how-to-setup-cluster/)

| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-etcd01    | 158.160.131.290   |
| srv-ubu-etcd02    | 158.160.131.291   |
| srv-ubu-etcd03    | 158.160.131.292   |

## Установка сервера Etcd. (На всех серверах)
```bash
wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/v3.5.0/etcd-v3.5.0-linux-amd64.tar.gz"
tar zxf etcd-v3.5.0-linux-amd64.tar.gz
mv etcd-v3.5.0-linux-amd64/etcd* /usr/bin/
chmod +x /usr/bin/etcd*
```
## Настройка кластера Etcd. (srv-ubu-etcd01)
```bash
vim /etc/etcd

ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://158.160.131.290:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://158.160.131.290:2380
ETCD_ADVERTISE_CLIENT_URLS=http://158.160.131.290:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://158.160.131.290:2380
ETCD_INITIAL_CLUSTER=etcd1=http://158.160.131.290:2380,etcd2=http://158.160.131.291:2380,etcd3=http://158.160.131.292:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

## Настройка кластера Etcd. (srv-ubu-etcd02)
```bash
vim /etc/etcd

ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://158.160.131.291:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://158.160.131.291:2380
ETCD_ADVERTISE_CLIENT_URLS=http://158.160.131.291:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://158.160.131.291:2380
ETCD_INITIAL_CLUSTER=etcd1=http://158.160.131.290:2380,etcd2=http://158.160.131.291:2380,etcd3=http://158.160.131.292:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

## Настройка кластера Etcd. (srv-ubu-etcd03)
```bash
vim /etc/etcd

ETCD_NAME=etcd1
ETCD_DATA_DIR=/var/lib/etcd
ETCD_LISTEN_CLIENT_URLS=http://158.160.131.292:2379,http://127.0.0.1:2379
ETCD_LISTEN_PEER_URLS=http://158.160.131.292:2380
ETCD_ADVERTISE_CLIENT_URLS=http://158.160.131.292:2379
ETCD_INITIAL_ADVERTISE_PEER_URLS=http://158.160.131.292:2380
ETCD_INITIAL_CLUSTER=etcd1=http://158.160.131.290:2380,etcd2=http://158.160.131.291:2380,etcd3=http://158.160.131.292:2380
ETCD_INITIAL_CLUSTER_STATE=new
ETCD_INITIAL_CLUSTER_TOKEN=etcd-cluster
```

## Настройка systemd сервиса. (На всех серверах)
```bash
vim /etc/systemd/system/etcd.service

[Unit]
Description=etcd

[Service]
Type=notify
EnvironmentFile=/etc/etcd
ExecStart=/usr/bin/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl enable etcd
service etcd status
```
>> Необходимо запустить как минимум 2 узла etcd в течение 50 секунд, иначе он вернет ошибку из-за тайм-аута.
>>> Проверка кластера.
```bash
ETCDCTL_API=3 etcdctl --endpoints=http://127.0.0.1:2379 member list

685732e85e851bdd, started, etcd1, http://158.160.131.290:2380, http://158.160.131.290:2379, false
8940390e3669b48e, started, etcd2, http://158.160.131.291:2380, http://158.160.131.291:2379, false
345721e85e456bzw, started, etcd3, http://158.160.131.292:2380, http://158.160.131.292:2379, false
```
## Добавление аунтификации. (На всех серверах)
```
etcdctl user add root
etcdctl auth enable
etcdctl --user root:rootpw auth disable
```
Установка с использованием tls - сертификатов: https://github.com/justmeandopensource/kubernetes/blob/master/kubeadm-external-etcd/2%20simple-cluster-tls.md
