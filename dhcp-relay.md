## Настройка DHCP relay на роутерах

## Описание
*не реализовано*

## Требования
* [Создание роутера для сегмента клиентов](client-router.md)

## Выполнение
1. На роутере "ClientRouter" добавим настройку DHCP relay:
    ```
    ip dhcp-relay add name=client-relay interface=ether2 dhcp-server=10.0.0.1 local-address=172.16.0.1 disabled=no
    ```
2. Проверим добавленную настройку:
    ```
    ip dhcp-relay print
    ```
3. На роутере "ServerRouter" добавим настройку DHCP relay:
    ```
    ip dhcp-relay add name=client-relay interface=ether1 dhcp-server=10.0.1.2 local-address=10.0.0.1 disabled=no
    ```
4. Проверим добавленную настройку:
    ```
    ip dhcp-relay print
    ```