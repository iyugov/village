# Создание коммутатора серверов баз данных

## Описание

Создаётся коммутатор для серверов баз данных.

## Требования

* [Создание сервера релпликации баз данных MySQL](mysql-replication.md)

## Выполнение

1. Создадим в GNS3 коммутатор ("Ethernet switch") с именем "DBServerSwitch".

2. Соединим порт `Ethernet0` хоста "DB1" с портом `Ethernet1` созданного коммутатора.

3. Соединим порт `Ethernet0` хоста "DB1R" с портом `Ethernet2` созданного коммутатора.

4. Соединим порт `Ethernet0` хоста "DB2" с портом `Ethernet3` созданного коммутатора.

5. Соединим порт `Ethernet0` созданного коммутатора с портом `Ethernet5` коммутатора `ServerSwitch`.
