# Установка кэширующего сервера имён

## Описание

Создаётся кэширующий сервер имён на базе Unbound, на сервере имён.

## Требования

* [Создание сервера имён](ns.md)
* [Создание хоста консольного клиента](client-cli.md)

## Выполнение

1. Установим пакет Unbound:

    ```sh
    sudo apt update
    sudo apt install unbound
    ```

2. Создадим и откроем в редакторе файл настроек Unbound:

    ```sh
    sudo vim /etc/unbound/unbound.conf.d/local.conf
    ```

3. В редакторе обеспечим следующее содержимое файла:

    ```config
    server:
        verbosity: 1
        interface: 127.0.0.1  
        port: 5353  
        access-control: 127.0.0.1/32 allow  
        cache-min-ttl: 3600  
        cache-max-ttl: 86400  
        prefetch: yes  
        val-permissive-mode: no
        auto-trust-anchor-file: "/var/lib/unbound/root.key"
        num-threads: 2  
        msg-cache-size: 64m  
        rrset-cache-size: 128m  
        key-cache-size: 32m  
        root-hints: "/var/lib/unbound/root.hints"
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

4. Обновим список корневых серверов для разрешения имён напрямую:

    ```sh
    sudo wget -O /var/lib/unbound/root.hints https://www.internic.net/domain/named.cache
    sudo chown unbound:unbound /var/lib/unbound/root.hints
    ```

5. Включим службу Unbound, перезапустим её и проверим её состояние:

    ```sh
    sudo systemctl enable unbound
    sudo systemctl restart unbound
    sudo systemctl status unbound
    ```

6. Создадим и откроем в редакторе файл настроек Bind9:

    ```sh
    sudo vim /etc/bind/named.conf.options
    ```

7. В редакторе изменим адрес для перенаправления с указанием порта, а также включим рекурсию, добавив или изменив следующие настройки (раздел `allow-query` оставим без изменения):

    ```config
        forwarders {
                127.0.0.1 port 5353;
        };

        recursion yes;

        allow-recursion {
                localhost;
                10.0.1.0/24;
                172.16.0.0/24;
        };
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

8. Перезапустим службу Bind9:

    ```sh
    sudo systemctl restart bind9
    ```

9. На хосте "Client 1" проверим работу кэширования запросами на разрешение имени хоста в интернете (например, google.com):

    ```sh
    dig @ns google.com
    ```

    Заметим время выполнения запроса ("Query time").

10. На том же хосте очистим локальный кэш запросов и выполним запрос повторно:

    ```sh
    sudo resolvectl flush-caches
    dig @ns google.com
    ```

    Несмотря на очистку локального кэша, время выполнения запроса должно быть существенно меньше предыдущего, т. к. работает кэш Ubound.
