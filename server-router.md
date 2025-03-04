# Создание роутера для сегмента серверов

## Описание

Не реализовано.

## Требования

* [Создание коммутатора для шлюза](gateway-switch.md)

## Выполнение

1. Создадим в GNS3 роутер Mikrotik RB450G, используя данные с сервера, по последней доступной версии прошивки, с именем "ServersRouter".
2. Соединим порт `ether1` этого роутера с портом `Ethernet1` коммутатора "GatewaySwitch".
3. Проведём настройку роутера:

    1) Установим пароль администратора (по умолчанию он пустой).
    2) Включим интерфейс `ether1`:

        ```mikrotik
        interface enable ether1
        ```

    3) Добавим статический IP-адрес 10.0.0.1/24 на интерфейсе `ether1`:

        ```mikrotik
        ip address add address=10.0.0.1/24 interface=ether1
        ```

    4) Проверим доступность роутера "Gateway":

        ```mikrotik
        ping 10.0.0.254
        ```

        Прервать проверку можно по `Ctrl`+`c`.  

    5) Включим интерфейс `ether2`:

        ```mikrotik
        interface enable ether2
        ```

    6) Добавим маршрут через роутер "Gateway":

        ```mikrotik
        ip route add gateway=10.0.0.254
        ```

    7) Добавим статический IP-адрес 10.0.1.1/24 на интерфейсе `ether2`:

        ```mikrotik
        ip address add address=10.0.1.1/24 interface=ether2
        ```

    8) Проверим таблицу маршрутизации:

        ```mikrotik
        ip route print
        ```

4. На роутере "Gateway" добавим маршрут в сеть серверов 10.0.1.0/24 через роутер серверов "ServerRouter":

    ```mikrotik
    ip route add dst-address=10.0.1.0/24 gateway=10.0.0.1
    ```
