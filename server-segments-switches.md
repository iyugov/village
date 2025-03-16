# Создание коммутаторов сегментов серверов

## Описание

Создаются коммутаторы для сегментов серверов: основного сегмента, сегмента администрирования и сегмента хранения. Выполняются переподключения хостов и коммутаторов.

## Требования

* [Создание сервера NFS](nfs.md)
* [Создание сервера Samba](smb.md)
* [Создание сервера репликации баз данных PostgreSQL](postgresql-replication.md)

## Выполнение

1. Создадим в GNS3 коммутатор ("Ethernet switch") с именем "AdminServerSwitch".

2. Соединим порт `Ethernet0` хоста "Access" с портом `Ethernet1` коммутатора "AdminServerSwitch".

3. Соединим порт `Ethernet0` хоста "CA" с портом `Ethernet2` коммутатора "AdminServerSwitch".

4. Создадим в GNS3 коммутатор ("Ethernet switch") с именем "BaseServerSwitch".

5. Соединим порт `Ethernet0` хоста "DHCP" с портом `Ethernet1` коммутатора "BaseServerSwitch".

6. Соединим порт `Ethernet0` хоста "NS" с портом `Ethernet2` коммутатора "BaseServerSwitch".

7. Создадим в GNS3 коммутатор ("Ethernet switch") с именем "StorageServerSwitch".

8. Соединим порт `Ethernet0` хоста "SMB" с портом `Ethernet1` коммутатора "StorageServerSwitch".

9. Соединим порт `Ethernet0` хоста "NFS" с портом `Ethernet2` коммутатора "StorageServerSwitch".

10. Соединим порт `Ethernet0` коммутатора "DBServerSwitch" с портом `Ethernet3` коммутатора "ServerSwitch".

11. Соединим порт `Ethernet0` коммутатора "AdminServerSwitch" с портом `Ethernet1` коммутатора "ServerSwitch".

12. Соединим порт `Ethernet0` коммутатора "BaseServerSwitch" с портом `Ethernet2` коммутатора "ServerSwitch".

13. Соединим порт `Ethernet0` коммутатора "StorageServerSwitch" с портом `Ethernet3` коммутатора "ServerSwitch".
