# Создание центра сертификации

## Описание

Создаётся хост центра сертификации (Certificate Authority) для выпуска и подписи сертификатов на базе Ubuntu Server 24.04 с RAID 1. Сервер имеет адрес 10.0.1.5/24.
Обеспечиваются базовые параметры безопасности.

## Требования

* [Создание сервера имён](ns.md)

## Выполнение

### Создание хоста для центра сертификации

1. Создадим виртуальную машину VirtualBox со следующими параметрами:

    * имя - "CA";
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

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet4` коммутатора "ServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.
    * Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.
    * Выберем вариант установки "Ubuntu Server".
    * Для сетевого интерфейса укажем настройки для IPv4 вручную:
        * подсеть - 10.0.1.0/24;
        * адрес - 10.0.1.5;
        * шлюз - 10.0.1.1;
        * сервер имён - 10.0.1.3;
        * домен поиска - "village".

7. Оставим настройку прокси пустой.

8. Для разметки накопителей выберем "Custom storage layout".

9. Создадим RAID 1:
    * выберем "Create software RAID (md)";
    * укажем имя "md0" и уровень 1 (зеркальный);
    * отметим устройства - 2 накопителя по 10 Гб.

10. Создадим LVM:
    * выберем "Create volume group (LVM)";
    * укажем имя "vg0" и расположение - "md0".

11. Сделаем накопитель размером 2 Гб загрузочным ("Use As Boot Device"). В свободном пространстве этого накопителя создадим раздел GPT ("Add GPT Partition") с размером по умолчанию, файловой системой ext4 и точкой монтирования "/boot".

12. В свободном пространстве группы логических томов "vg0" создадим логический том ("Create Logical Volume") с именем "lv-0", размером по умолчанию, файловой системой ext4 и точкой монтирования "/".

13. Проверим разметку, завершим её, согласимся на изменения.

14. Зададим отображаемое имя пользователя "User", имя сервера "ca", имя пользователя "user"; зададим пароль пользователя.

15. Пропустим настройку Ubuntu Pro.

16. Отметим установку сервера OpenSSH.

17. Дождёмся окончания установки. Перезагрузим сервер. Авторизуемся.

18. Обновим пакеты:

    ```sh
    sudo apt update
    sudo apt upgrade
    sudo apt dist-upgrade
    ```

### Настройка центра сертификации

1. Откроем файл настроек OpenSSL:

    ```sh
    sudo vim /etc/ssl/openssl.cnf
    ```

2. В файле после строки `[ CA_default ]` изменим строку, начинающуюся с `dir`, следующим образом:

    ```config
    [ CA_default ]

    dir =    ./ca    # Where everything is kept
    ```

3. В файле после строки `[ req_distinguished_name ]` изменим значения следующих строк (при отсутствии каких-то строк — допшем их, убедившись, что перед ними не поставлен знак «#»):

    ```config
    [ req_distinguished_name ]

    countryName_default               = RU
    ...
    stateOrProvinceName_default       = Тверская область
    ...
    0.organizationName_default        = Village
    ...
    0.organizationalUnitName_default  = УЦ
    ...
    commonName_default                = village
    ...
    emailAddress_default              = admin@village
    ```

4. В файле после строки `[ v3_ca ]` найдём строку, начинающуюся с «# keyUsage», и уберём из её начала символ «#»:

    ```config
    # Key usage: this is typical for a CA certificate. However since it will
    # prevent it being used as an test self-signed certificate it is best
    # left out by default.

    keyUsage = cRLSign, keyCertSign
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

5. Создадим каталог удостоверяющего центра:

    ```sh
    sudo mkdir /etc/ssl/ca
    ```

6. Перейдём в созданный каталог:

    ```sh
    cd /etc/ssl/ca
    ```

7. Создадим дополнительные каталоги:

    ```sh
    sudo mkdir newcerts certs crl private requests
    ```

8. Создадим файлы индекса базы данных:

    ```sh
    sudo touch index.txt index.txt.attr
    ```

9. Создадим файл текущего серийного номера:

    ```sh
    sudo bash -c 'echo 1000 > serial'
    ```

10. Сгенерируем сертификат ключа:

    Начнём генерацию ключа:

    ```sh
    sudo openssl req -new -newkey ed25519 -sha256 -days 3653 -nodes -x509 -extensions v3_ca -utf8 -set_serial 0 -out /etc/ssl/ca/ca_cert.pem -keyout /etc/ssl/ca/private/ca_key.pem
    ```

    Введём пароль ключа, заданный ранее.

    На запросы всех параметров ответим нажатием Enter, т. к. все параметры соответствуют параметрам, заданным в настройках OpenSSL по умолчанию:

    ```config
    Country Name (2 letter code) [RU]:
    State or Province Name (full name) [Тверская область]:
    Locality Name (eg, city) []:
    Organization Name (eg, company) [Village]:
    Organizational Unit Name (eg, section) [УЦ]:
    Common Name (e.g. server FQDN or YOUR name) [village]:
    Email Address [admin@village]:
    ```

    Сгенерированный сертификат находится в файле /etc/ssl/ca/ca_cert.pem и действует 10 лет (3653 дня).

11. Зададим права доступа к каталогу удостоверяющего центра - на чтение для всех, на запись только для владельца (суперпользователя):

    ```sh
    sudo chmod -R 755 /etc/ssl/ca
    ```

12. Зададим права доступа к каталогам закрытых ключей и запросов - только для суперпользователя:

    ```sh
    sudo chmod -R 700 /etc/ssl/ca/{private,requests}
    ```

#### Добавление записей на сервер имён

1. На сервере "NS" откроем файл локальной прямой зоны:

    ```sh
    sudo vim /etc/bind/db.village
    ```

2. Добавим в файл следующую строку:

    ```config
    ca      IN      A       10.0.1.5
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
    5       IN     PTR    ca.village.
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

### Настройка безопасности сервера

#### Настройка доступа на сервер по ключу

1. На сервере "CA" откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_ca`.
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на целемой сервер:

    ```sh
    ssh-copy-id -i ~/.ssh/id_ca user@ca
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_ca user@ca
    ```

    Авторизация должна пройти без запроса пароля.  

9. На сервере "CA" откроем файл настроек сервера SSH:

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

    Добавим в файл следующий раздел:  

    ```config
    Host ca
      Hostname ca
      Port 22
      IdentityFile ~/.ssh/id_ca
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер центра сертификации с учётом настроек:

    ```sh
    ssh ca
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "CA" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

#### Настройка fail2ban на сервере

1. На сервере "CA" установим *fail2ban*:

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

#### Настройка отправки уведомлений о блокировках доступа на сервер через локальный почтовый сервер

1. На сервере "CA" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `ca`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@ca
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
    destemail = user@ca
    sender = fail2ban@ca
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим отправку сообщений при блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.
