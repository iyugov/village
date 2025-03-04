# Настройка перенаправления портов для сервера доступа

## Описание

Обеспечивается перенаправление портов для доступа из внешней сети через роутер "Gateway" на сервер доступа.

## Требования

* [Создание хоста для сервера доступа](access.md)

## Выполнение

1. На роутере "Gateway" выполнить перенаправление портов для SSH к серверу "access" через роутер "ServerRouter":

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=4224 action=dst-nat to-addresses=10.0.0.1 to-ports=4224
    ```

2. На роутере "ServerRouter" выполнить перенаправление портов для SSH к серверу "access":

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=4224 action=dst-nat to-addresses=10.0.1.3 to-ports=22
    ```

3. На роутере "Gateway" выяснить его внешний IP-адрес (на интерфейсе `ether1`):

    ```mikrotik
    ip address print
    ```

4. С компьютера в локальной сети гипервизора подключимся к серверу "access" (изменим адрес 192.168.122.101 на выясненный адрес роутера "Gateway"):

    ```sh
    ssh user@192.168.122.101 -p 4224
    ```

    также проверим подключение к роутеру "Gateway":  

    ```sh
    telnet 192.168.1.101
    ```

    Подключение должно состояться.

5. В настройках хоста "Access" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.
