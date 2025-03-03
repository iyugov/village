# Настройка DHCP relay на роутерах

## Описание

Обеспечивается DHCP relay на роутерах клиентов и серверов, для обеспечения выдачи клиентам адресов в подсети 172.16.0.0/16 DHCP-сервером, находящимся в другой подсети - 10.0.1.0/24.

## Требования

* [Создание роутера для сегмента клиентов](client-router.md)

## Выполнение

1. На роутере "ClientRouter" добавим настройку DHCP relay:

    ```mikrotik
    ip dhcp-relay add name=client-relay interface=ether2 dhcp-server=10.0.0.1 local-address=172.16.0.1 disabled=no
    ```

2. Проверим добавленную настройку:

    ```mikrotik
    ip dhcp-relay print
    ```

3. На роутере "ServerRouter" добавим настройку DHCP relay:

    ```mikrotik
    ip dhcp-relay add name=client-relay interface=ether1 dhcp-server=10.0.1.2 local-address=10.0.0.1 disabled=no
    ```

4. Проверим добавленную настройку:

    ```mikrotik
    ip dhcp-relay print
    ```
