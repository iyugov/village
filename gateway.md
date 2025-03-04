# Создание роутера для шлюза

## Описание

Создаётся роутер Mikrotik, служащий шлюзом с NAT между виртуальной сетью проекта и сетью базовой системы. Его адрес во внутренней сети - 10.0.0.254/24, во внешней - выдаётся DHCP-сервером в сети базовой системы, шлюз в интернет имеет адрес 192.168.1.1.

## Требования

* [Старт проекта](start.md)

## Выполнение

1. Создадим в GNS3 роутер Mikrotik RB450G, используя данные с сервера, по последней доступной версии прошивки, с именем "Gateway".
2. Соединим порт `ether1` роутера с облаком "ISP".
3. Проведём настройку роутера:
    1) Установим пароль администратора (по умолчанию он пустой).
    2) Включим интерфейс `ether1`:

        ```mikrotik
        interface enable ether1
        ```

    3) Проверим статус DHCP-клиента:

        ```mikrotik
        ip dhcp-client print
        ```

        На интерфейсе `ether1` должен работать DHCP-клиент. Если это не так, то включим его:  

        ```mikrotik
        ip dhcp-client add interface=ether1 use-peer-dns=yes add-default-route=yes
        ```

    4) Включим интерфейс `ether2`:

        ```mikrotik
        interface enable ether2
        ```

    5) Добавим маршрут через шлюз провайдера (шлюз системы гипервизора):

        ```mikrotik
        ip route add gateway=192.168.1.1
        ```

    6) Добавим статический IP-адрес 10.0.0.254/24 на интерфейсе `ether2`:

        ```mikrotik
        ip address add address=10.0.0.254/24 interface=ether2
        ```

    7) Включим NAT из одной сети в другую:

        ```mikrotik
        ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
        ```

    8) Проверим таблицу маршрутизации:

        ```mikrotik
        ip route print
        ```

    9) Проверим статус NAT:

        ```mikrotik
        ip firewall net print
        ```
