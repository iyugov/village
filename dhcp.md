# Создание DHCP-сервера

## Описание

Создаётся DHCP-сервер на базе Ubuntu Server 24.04 с RAID 1. Сервер имеет адрес 10.0.1.4/24 и управляет адресами в подсети 172.16.0.0/16.
Обеспечиваются базовые параметры безопасности.
Настраивается DHCP relay на роутерах клиентов и серверов, для обеспечения выдачи клиентам адресов в подсети 172.16.0.0/16 DHCP-сервером, находящимся в другой подсети - 10.0.1.0/24.

## Требования

* [Создание сервера имён](ns.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:
    * имя - "DHCP";
    * ОС - Linux, Ubuntu (64-bit);
    * объём оперативной памяти - 2048 Мб, процессоры - 1;
    * накопители:
    * SATA, 2 Гб;
    * SATA, 10 Гб;
    * SATA, 10 Гб.
    * оптический привод для установочного образа;
    * сетевой адаптер: имеет тип Generic Driver с именем UDPTunnel.

2. Добавим виртуальную машину в GNS3 через шаблон.

3. Установим управление GNS3 для сетевых интерфейсов приложения и зададим число сетевых адаптеров - 1.

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet3` коммутатора "ServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.

7. Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.

8. Выберем вариант установки "Ubuntu Server".

9. Для сетевого интерфейса укажем настройки для IPv4 вручную:
    * подсеть - 10.0.1.0/24;
    * адрес - 10.0.1.3;
    * шлюз - 10.0.1.1;
    * сервер имён - 10.0.1.4.

10. Оставим настройку прокси пустой.

11. Для разметки накопителей выберем "Custom storage layout".

12. Создадим RAID 1:
    * выберем "Create software RAID (md)";
    * укажем имя "md0" и уровень 1 (зеркальный);
    * отметим устройства - 2 накопителя по 10 Гб.

13. Создадим LVM:
    * выберем "Create volume group (LVM)";
    * укажем имя "vg0" и расположение - "md0".

14. Сделаем накопитель размером 2 Гб загрузочным ("Use As Boot Device"). В свободном пространстве этого накопителя создадим раздел GPT ("Add GPT Partition") с размером по умолчанию, файловой системой ext4 и точкой монтирования "/boot".

15. В свободном пространстве группы логических томов "vg0" создадим логический том ("Create Logical Volume") с именем "lv-0", размером по умолчанию, файловой системой ext4 и точкой монтирования "/".

16. Проверим разметку, завершим её, согласимся на изменения.

17. Зададим отображаемое имя пользователя "User", имя сервера "dhcp", имя пользователя "user"; зададим пароль пользователя.

18. Пропустим настройку Ubuntu Pro.

19. Отметим установку сервера OpenSSH.

20. Дождёмся окончания установки. Перезагрузим сервер. Авторизуемся.

21. Обновим пакеты:

    ```sh
    sudo apt update
    sudo apt upgrade
    sudo apt dist-upgrade
    ```

### Настройка DHCP-сервера

1. На DHCP-сервер установим соответствующий пакет:

    ```sh
    sudo apt install isc-dhcp-server
    ```

2. Отобразим список сетевых интерфейсов:

    ```sh
    ip address
    ```

    Определим имя сетевого интерфейса, подключённого к роутеру "ServerRouter", т. е. имеющего IP-адрес 10.0.1.4.

3. Откроем файл настроек DHCP-сервера по умолчанию:

    ```sh
    sudo vim /etc/default/isc-dhcp-server
    ```

4. В файле найдём строку `INTERFACESv4=""` и впишем в кавычки определённое ранее имя интерфейса (пример для `enp0s3`):

    ```config
    INTERFACESv4="enp0s3"
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

5. Откроем файл настроек DHCP-сервера:

    ```sh
    sudo vim /etc/dhcp/dhcpd.conf
    ```

    В файле найдём строку `#authoritative;` и раскомментируем её для настройки сервера как авторитетного:  

    ```config
    authoritative;
    ```

6. Добавим в файл секцию описания совместной зоны для текущей подсети сервера и клиентской подсети:

    ```config
    shared-network combined-pools {
      subnet 10.0.1.0 netmask 255.255.0.0 {
      }
      subnet 172.16.0.0 netmask 255.255.0.0 {
        range 172.16.0.101 172.16.0.200;
        option routers 172.16.0.1;
        option domain-name-servers 10.0.1.3;
      }
    }
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

7. Запустим сервер:

    ```sh
    sudo systemctl start isc-dhcp-server
    ```

8. Проверим статус сервера:

    ```sh
    sudo systemctl status isc-dhcp-server
    ```

    Сервер должен быть активен и запущен (`active (running)`).

#### Добавление записей на сервер имён

1. На сервере "NS" откроем файл локальной прямой зоны:

    ```sh
    sudo vim /etc/bind/db.village
    ```

2. Добавим в файл следующую строку:

    ```config
    dhcp    IN      A       10.0.1.4
    ```

    Увеличим значение параметра `Serial` на 1.
    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. Проверим конфигурацию зоны:

    ```sh
    sudo named-checkzone village /etc/bind/db.village
    ```

4. На сервере "NS" откроем файл локальной обратной зоны:

    ```sh
    sudo vim /etc/bind/db.10.0.1
    ```

5. Добавим в файл следующую строку:

    ```config
    4       IN     PTR    dhcp.village.
    ```

    Увеличим значение параметра `Serial` на 1.
    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

6. Проверим конфигурацию зоны:

    ```sh
    sudo named-checkzone 1.0.10.in-addr.arpa /etc/bind/db.10.0.1
    ```

7. Перезапустим *bind*:

    ```sh
    sudo systemctl restart bind9
    ```

### Настройка безопасности сервера DHCP

#### Настройка доступа на сервер DHCP по ключу

1. На сервере "DHCP" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

2. Установим следующие значения параметров:

    ```config
    PermitRootLogin no
    StrictModes yes
    MaxAuthTries 6
    PubkeyAuthentication yes
    PasswordAuthentication yes
    AllowUsers user
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

4. Установим правило межсетевого экрана для OpenSSH:

    ```sh
    sudo ufw allow OpenSSH
    ```

5. Включим межсетевой экран:

    ```sh
    sudo ufw enable
    ```

6. На сервере "Access" сгеренируем ключ:

    ```sh
    ssh-keygen -t ed25519 -C "admin@village"
    ```

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_dhcp`.
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на сервер "DHCP":

    ```sh
    ssh-copy-id -i ~/.ssh/id_dhcp user@dhcp.village
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_dhcp user@dhcp.village
    ```

    Авторизация должна пройти без запроса пароля.  

9. На сервере "DHCP" откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

10. Выключим авторизацию по паролю:

    ```config
    PasswordAuthentication no
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

11. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

12. На сервере "Access" создадим и откроем файл настроек подключений по SSH:

    ```sh
    vim ~/.ssh/config
    ```

    Обеспечим следующие содержание файла:  

    ```config
    Host dhcp
    Hostname dhcp.village
    Port 22
    IdentityFile ~/.ssh/id_dhcp
    IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер DHCP с учётом настроек:

    ```sh
    ssh dhcp
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "DHCP" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

#### Настройка fail2ban на сервере DHCP

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

#### Настройка отправки уведомлений о блокировках доступа на сервер DHCP через локальный почтовый сервер

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

### Настройка DHCP relay на роутерах клиентов и серверов

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
    ip dhcp-relay add name=client-relay interface=ether1 dhcp-server=10.0.1.4 local-address=10.0.0.1 disabled=no
    ```

4. Проверим добавленную настройку:

    ```mikrotik
    ip dhcp-relay print
    ```
