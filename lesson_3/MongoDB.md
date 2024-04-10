# Первичная настройка и установка сервера MongoDB (.tgz)
**Установка будет выполнена из Tarball по руководству:** [MongoDB Tarball](https://www.mongodb.com/docs/manual/tutorial/install-mongodb-on-ubuntu-tarball/)

Данный способ выбран из-за проблем, которые могут возникать из-за служб ИБ (Информационной Безопастности).  
Например требований к проверке исходного кода или запрета подключения дополнительных репозиториев к серверу.

## Первичная настройка параметров ОС для MongoDB.
**Согласно руководству необходимо выполнить выключение больших страниц ОС:** [MongoDB THP](https://www.mongodb.com/docs/manual/tutorial/transparent-huge-pages/)
```bash
cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
sudo vim /etc/default/grub

# Добавляем строку

GRUB_CMDLINE_LINUX="transparent_hugepage=never"
sudo grub-mkconfig -o /boot/grub/grub.cfg
```
**Так же согласно руководству необходимо выставить лимиты ОС:** [MongoDB ulimit Settings](https://www.mongodb.com/docs/manual/reference/ulimit/)
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
Для применения параметров необходимо выполнить **перезагрузку системы**.
