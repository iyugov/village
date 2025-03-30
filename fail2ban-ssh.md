# Настройка fail2ban для сервера SSH

## Описание

Настраивается fail2ban для службы SSH, блокирующий многократные попытки авторизации.

## Требования

* Хост на Ubuntu Server, с настроенным сервером SSH, принимающем подключения на порту SSH_PORT непосредственно или через перенаправление портов.
* Клиент SSH, имеющий сетевой доступ к серверу.

## Выполнение

1. На хосте, которая настраивается установим *fail2ban*:

    ```sh
    sudo apt update
    sudo apt install fail2ban
    ```

2. Создадим файл локальной конфигурации копированием имеющегося:

    ```sh
    sudo cp /etc/fail2ban/jail.{conf,local}
    ```

3. Откроем файл локальной конфигурации:

    ```sh
    sudo vim /etc/fail2ban/jail.local
    ```

4. Приведём файл к следующему виду:

    ```config
    [sshd]
    enabled = true
    port = 22
    logpath = %(sshd_log)s
    backend = %(sshd_backend)s
    maxretry = 3
    findtime = 10m
    bantime = 30m
    bantime.increment = true
    bantime.ratio = 1
    bantime.factor = 2
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  

5. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

6. Проверим работу *fail2ban*: попытаемся подключиться с другого хоста (SSH-клиента) по SSH и авторизоваться с неверным именем пользователя и/или неверным паролем. После трёх неудачных попыток авторизации в следующих соединениях с другого клиента должно быть отказано даже с правильным именем пользователя (вместо `SERVER_NAME` будет указан IP-адрес или имя хоста сервера SSH, а вместо `SERVER_PORT` порт сервера SSH):

    ```sh
    ssh: connect to host SERVER_NAME port SERVER_PORT: Connection refused
    ```

7. *Для разблокировки заблокированного хоста SSH-клиента по IP-адресу можно использовать следующую команду непосредственно на настроенном хосте (вместо `CLIENT_IP` нужно указать IP-адрес хоста):*

    ```sh
    sudo fail2ban-client set sshd unbanip CLIENT_IP
    ```

8. *Список заблокированных клиентов можно посмотреть следующим образом:*

    ```sh
    sudo fail2ban-client status sshd
    ```
