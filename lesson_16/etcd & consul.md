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

ETCD_NAME=etcd2
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

ETCD_NAME=etcd3
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

# Установка и настройка кластера Consul.
> Установка выполнена через **сервис:** YandexCloud.

>> Установка будет выполнена из *.deb пакетов по **руководству:** [Consul Install](https://developer.hashicorp.com/consul/docs/install#precompiled-binaries)
>>> Установка кластера будет выполнена по **руководству**: [Consul Cluster](https://developer.hashicorp.com/consul/tutorials/production-vms/deployment-guide?productSlug=consul&tutorialSlug=production-deploy&tutorialSlug=deployment-guide)

| Cервера  | ip | 
| ----------- | ----------- |
| srv-ubu-consul01    | 158.160.131.290   |
| srv-ubu-consul02    | 158.160.131.291   |
| srv-ubu-consul03    | 158.160.131.292   |

## Установка сервера Consul. (На всех серверах)
```bash
CONSUL_VER="1.17.1"
wget https://releases.hashicorp.com/consul/${CONSUL_VER}/consul_${CONSUL_VER}_linux_amd64.zip
unzip consul_*_linux_amd64.zip -d /usr/bin/
consul -v
```

## Настройка кластера Consul. (На всех серверах)
```bash
useradd -r -c 'Consul DCS service' consul
mkdir -p /var/lib/consul /etc/consul.d
chown consul:consul /var/lib/consul /etc/consul.d
chmod 775 /var/lib/consul /etc/consul.d
CONSUL_SERVER1=srv-ubu-consul01
CONSUL_SERVER1=srv-ubu-consul02
CONSUL_SERVER1=srv-ubu-consul03
```
## Генерация ключей. (srv-ubu-consul01)
```
consul keygen
CONSUL_TOKEN=wHFWVHTstpfh08ZflUs4FD2FAMueraoCN2LyqmeLxV0=
```
## Конфигурационный файл. (На всех серверах)
```bash
cat <<EOF > /etc/consul.d/config.json
{
     "bind_addr": "0.0.0.0",
     "bootstrap_expect": 3,
     "client_addr": "0.0.0.0",
     "datacenter": "dc1",
     "node_name": "$(hostname)",
     "data_dir": "/var/lib/consul",
     "domain": "consul",
     "enable_local_script_checks": true,
     "dns_config": {
         "enable_truncate": true,
         "only_passing": true
     },
     "enable_syslog": true,
     "encrypt": "${CONSUL_TOKEN}",
     "leave_on_terminate": true,
     "log_level": "INFO",
     "rejoin_after_leave": true,
     "retry_join": [
         "${CONSUL_SERVER1}",
         "${CONSUL_SERVER2}",
         "${CONSUL_SERVER3}"
     ],
     "server": true,
     "start_join": [
         "${CONSUL_SERVER1}",
         "${CONSUL_SERVER2}",
         "${CONSUL_SERVER3}"
     ],
    "ui_config": { "enabled": true }
}
EOF

cat /etc/consul.d/config.json
consul validate /etc/consul.d
```
## Настройка systemd сервиса. (На всех серверах)
```bash
vi /etc/systemd/system/consul.service

[Unit]
Description=Consul Service Discovery Agent
Documentation=https://www.consul.io/
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=consul
Group=consul
ExecStart=/usr/bin/consul agent \
    -config-dir=/etc/consul.d
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGINT
TimeoutStopSec=5
Restart=on-failure
SyslogIdentifier=consul

[Install]
WantedBy=multi-user.target

systemctl daemon-reload
systemctl start consul
systemctl enable consul
systemctl status consul
```
## Проверка кластера.
```bash
consul members -detailed
```
>Также у нас должен быть доступен веб-интерфейс по адресу: http://<IP-адрес любого сервера консул>:8500.

>Перейдя по нему, мы должны увидеть страницу со статусом нашего кластера.

## Аутентификация Consul. (На всех серверах)
```bash
vi /etc/consul.d/config.json

{ 
  ...
    "ui_config": { "enabled": true },
    "acl": {
        "enabled": true,
        "default_policy": "deny",
        "enable_token_persistence": true
    }
}

consul validate /etc/consul.d/config.json

systemctl restart consul
```

## Работа с токенами.
```bash
consul acl bootstrap
export CONSUL_HTTP_TOKEN=59ac7fa8-dca6-e066-ff33-0bf9bb6f466a

#создать новый токен
consul acl token create -policy-name global-management
#список токенов
consul acl token list
#удаление токена
consul acl token delete -id 54b5f2bb-1a57-3884-f0ea-1284f84186f5
```

## Настройка политики для DNS. (политика на srv-ubu-consul01 ключ DNS на всех серверах)
> После включения ACL наша система перестанет отвечать на запросы DNS.
>> Это связано с политикой блокировки по умолчанию.
>> Для разрешения запросов мы должны создать политику с разрешением данных запросов, получить для ее токен и применить данный токен на всех нодах консула.
```bash
cd /etc/consul.d/
vi dns-request-policy.txt

node_prefix "" {
  policy = "read"
}
service_prefix "" {
  policy = "read"
}

consul acl policy create -name "dns-requests" -rules @dns-request-policy.txt
```
> Создаем токен безопасности на основе политики:
```bash
consul acl token create -description "Token for DNS Requests" -policy-name dns-requests

#на всех нодах
export CONSUL_HTTP_TOKEN=59ac7fa8-dca6-e066-ff33-0bf9bb6f466a
consul acl set-agent-token default 42bd65e5-42a5-356b-a81b-60eff20f657
```
>где 59ac7fa8-dca6-e066-ff33-0bf9bb6f466a — ранее созданный токен (SecretID) для получения доступа к управлению консулом.
>где 42bd65e5-42a5-356b-a81b-60eff20f657 — тот SecretID, который мы получили на основе политики, разрешающей запросы DNS.
