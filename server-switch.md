# Создание коммутатора для серверов

## Описание

Настраивается коммутатор в сети серверов. Через него серверы подключаются к роутеру серверов.

## Требования

* [Создание DHCP-сервера](dhcp.md)

## Выполнение

1. Создадим в GNS3 коммутатор ("Ethernet switch") с именем "ServerSwitch".

2. Уберём соединение порта `Ethernet0` хоста "DHCP" и порта `ether2` роутера "ServerRouter".

3. Соединим порт `Ethernet0` этого коммутатора с портом `ether2` роутера "ServerRouter".

4. Соединим порт `Ethernet1` этого коммутатора с портом `Ethernet0` хоста "DHCP".
