# Настройка безопасности сервера доступа и сервера DHCP

## Описание

Настраивается доступ на серверы по ключу, параметры безопасности подключения, блокировка повторных попыток подключения, отправка уведомлений о блокировках на локальные почтовые серверы.

## Требования

* [Создание хоста для сервера доступа](access.md)

## Выполнение

### Настройка доступа на сервер доступа по ключу

1. На хосте, с которого доступен сервер "Access" и с которого собираемся подключаться по SSH к этому серверу, сгеренируем ключ (вместо `email` укажем адрес электронной почты администратора):

    ```sh
    ssh-keygen -t ed25519 -C "email"
    ```

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_village`.
    Зададим пустой пароль.

2. Выполним копирование сгенерированного ключа на сервер "access" (укажем файл ключа и внешний адрес роутера "Gateway"):

    ```sh
    ssh-copy-id -i ~/.ssh/id_village -p 4224 user@192.168.122.201
    ```

3. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_village user@192.168.122.201 -p 4224
    ```

    Авторизация должна пройти без запроса пароля.

4. На сервере "access" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

5. Установим следующие значения параметров:

    ```config
    PermitRootLogin no
    StrictModes yes
    MaxAuthTries 6
    PubkeyAuthentication yes
    PasswordAuthentication no
    AllowUsers user
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

6. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

7. Установим правило межсетевого экрана для нового порта SSH:

    ```sh
    sudo ufw allow OpenSSH
    ```

8. Включим межсетевой экран:

    ```sh
    sudo ufw enable
    ```

### Настройка доступа на сервер DHCP по ключу

1. В настройках хоста "Access" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

2. На сервере "DHCP" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

3. Установим следующие значения параметров:

    ```config
    PermitRootLogin no
    StrictModes yes
    MaxAuthTries 6
    PubkeyAuthentication yes
    PasswordAuthentication yes
    AllowUsers user
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

4. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

5. Установим правило межсетевого экрана для OpenSSH:

    ```sh
    sudo ufw allow OpenSSH
    ```

6. Включим межсетевой экран:

    ```sh
    sudo ufw enable
    ```

7. На сервере "Access" сгеренируем ключ:

    ```sh
    ssh-keygen -t ed25519 -C "admin@village"
    ```

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_dhcp`.
    Зададим пустой пароль.

8. Выполним копирование сгенерированного ключа на сервер "DHCP":

    ```sh
    ssh-copy-id -i ~/.ssh/id_dhcp user@10.0.1.2
    ```

9. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_dhcp user@10.0.1.2
    ```

    Авторизация должна пройти без запроса пароля.  

10. На сервере "DHCP" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

11. Выключим авторизацию по паролю:

    ```config
    PasswordAuthentication no
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

12. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

13. На сервере "Access" создадим и откроем файл настроек подключений по SSH:

    ```sh
    vim ~/.ssh/config
    ```

    Обеспечим следующие содержание файла:  

    ```config
    Host dhcp
    Hostname 10.0.1.2
    Port 22
    IdentityFile ~/.ssh/id_dhcp
    IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

14. Выполним вход на сервер DHCP с учётом настроек:

    ```sh
    ssh dhcp
    ```

    Авторизация должна пройти автоматически.

15. В настройках хоста "DHCP" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

### Настройка fail2ban на сервере доступа

1. На сервере "Access" установим *fail2ban*:

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

6. Проверим работу *fail2ban*: попытаемся подключиться с клиента по SSH и авторизоваться с неверным именем пользователя и/или неверным паролем. После трёх неудачных попыток авторизации в следующих соединениях с клиента должно быть отказано даже с правильным именем пользователя:

    ```sh
    ssh: connect to host 192.168.122.201 port 4224: Connection refused
    ```

7. *Для разблокировки заблокированного клиента по IP-адресу можно использовать следующую команду непосредственно на сервере (пример для адреса 192.168.122.1):*

    ```sh
    sudo fail2ban-client set sshd unbanip 192.168.122.1
    ```

8. *Список заблокированных клиентов можно посмотреть следующим образом:*

    ```sh
    sudo fail2ban-client status sshd
    ```

### Настройка отправки уведомлений о блокировках доступа на сервер доступа через локальный почтовый сервер

1. На сервере "Access" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `access`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@access
    ```

5. Проверим получение локальной почты:

    ```sh
    mail
    ```

    В отображённом списке сообщений должно быть сообщение с тестовой темой. На запрос `?` нажмём `Enter`. Откроется само сообщение. Нажмём `q` для выхода из программы просмотра почты. При выходе сообщение удаляется из входящей почты и сохраняется в файле `mbox` в домашнем каталоге пользователя.  

6. *Для более удобного управления сообщениями можно установить программу `mutt`*:

    ```sh
    sudo apt install mutt
    ```

7. Откроем файл локальной конфигурации:

    ```sh
    sudo vim /etc/fail2ban/jail.local
    ```

8. Добавим в секцию `[sshd]` следующие строки:

    ```config
    destemail = user@access
    sender = fail2ban@access
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим работу *fail2ban*: спровоцируем блокировку при серии подключений с клиента с неправильными учётными данными и проверим локальную почту на сервере. В почте должно быть сообщение о блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.

### Настройка fail2ban на сервере DHCP

1. На сервере "DHCP" установим *fail2ban*:

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

6. Проверим работу *fail2ban* по блокировке неверных авторизаций.

### Настройка отправки уведомлений о блокировках доступа на сервер DHCP через локальный почтовый сервер

1. На сервере "DHCP" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `dhcp`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@dhcp
    ```

5. Проверим получение локальной почты:

    ```sh
    mail
    ```

    В отображённом списке сообщений должно быть сообщение с тестовой темой. На запрос `?` нажмём `Enter`. Откроется само сообщение. Нажмём `q` для выхода из программы просмотра почты. При выходе сообщение удаляется из входящей почты и сохраняется в файле `mbox` в домашнем каталоге пользователя.

6. *Для более удобного управления сообщениями можно установить программу `mutt`*:

    ```sh
    sudo apt install mutt
    ```

7. Откроем файл локальной конфигурации:

    ```sh
    sudo vim /etc/fail2ban/jail.local
    ```

8. Добавим в секцию `[sshd]` следующие строки:

    ```config
    destemail = user@dhcp
    sender = fail2ban@dhcp
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим отправку сообщений при блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.
