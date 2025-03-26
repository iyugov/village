# Создание объектного хранилища

## Описание

Создаётся сервер объектного хранилища MinIO на базе Ubuntu Server 24.04 с RAID 1. Сервер имеет адрес 10.0.1.13/24.
Обеспечиваются базовые параметры безопасности и мониторинга.

## Требования

* [Создание сервера мониторинга](monitoring.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:
    * имя - "Objects";
    * ОС - Linux, Ubuntu (64-bit);
    * объём оперативной памяти - 2048 Мб, процессоры - 1;
    * накопители:
    * SATA, 2 Гб;
    * SATA, 20 2б;
    * SATA, 20 2б.
    * оптический привод для установочного образа;
    * сетевой адаптер: имеет тип Generic Driver с именем UDPTunnel.

2. Добавим виртуальную машину в GNS3 через шаблон.

3. Установим управление GNS3 для сетевых интерфейсов приложения и зададим число сетевых адаптеров - 1.

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet3` коммутатора "StorageServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.

7. Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.

8. Выберем вариант установки "Ubuntu Server".

9. Для сетевого интерфейса укажем настройки для IPv4 вручную:
    * подсеть - 10.0.1.0/24;
    * адрес - 10.0.1.13;
    * шлюз - 10.0.1.1;
    * сервер имён - 10.0.1.3;
    * домен поиска - "village".

10. Оставим настройку прокси пустой.

11. Для разметки накопителей выберем "Custom storage layout".

12. Создадим RAID 1:
    * выберем "Create software RAID (md)";
    * укажем имя "md0" и уровень 1 (зеркальный);
    * отметим устройства - 2 накопителя по 20 Гб.

13. Создадим LVM:
    * выберем "Create volume group (LVM)";
    * укажем имя "vg0" и расположение - "md0".

14. Сделаем накопитель размером 2 Гб загрузочным ("Use As Boot Device"). В свободном пространстве этого накопителя создадим раздел GPT ("Add GPT Partition") с размером по умолчанию, файловой системой ext4 и точкой монтирования "/boot".

15. В свободном пространстве группы логических томов "vg0" создадим логический том ("Create Logical Volume") с именем "lv-0", размером по умолчанию, файловой системой ext4 и точкой монтирования "/".

16. Проверим разметку, завершим её, согласимся на изменения.

17. Зададим отображаемое имя пользователя "User", имя сервера "objects", имя пользователя "user"; зададим пароль пользователя.

18. Пропустим настройку Ubuntu Pro.

19. Отметим установку сервера OpenSSH.

20. Дождёмся окончания установки. Перезагрузим сервер. Авторизуемся.

21. Обновим пакеты:

    ```sh
    sudo apt update
    sudo apt upgrade
    sudo apt dist-upgrade
    ```

#### Добавление записей на сервер имён

1. На сервере "NS" откроем файл локальной прямой зоны:

    ```sh
    sudo vim /etc/bind/db.village
    ```

2. Добавим в файл следующую строку:

    ```config
    objects IN      A       10.0.1.13
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
    13      IN     PTR    objects.village.
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

1. На сервере "Objects" откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_objects`.
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на целевой сервер:

    ```sh
    ssh-copy-id -i ~/.ssh/id_objects user@objects
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_objects user@objects
    ```

    Авторизация должна пройти без запроса пароля.  

9. На сервере "NFS" откроем файл настроек сервера SSH:

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
    Host objects
      Hostname objects
      Port 22
      IdentityFile ~/.ssh/id_objects
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер с учётом настроек:

    ```sh
    ssh objects
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "Objects" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

#### Настройка fail2ban на сервере

1. На сервере "SMB" установим *fail2ban*:

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

1. На сервере "NFS" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `objects`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@objects
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
    destemail = user@objects
    sender = fail2ban@objects
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим отправку сообщений при блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.

### Настройка сервера MinIO

1. В любом доступном веб-браузере по адресу <hhttps://min.io/docs/minio/linux/operations/install-deploy-manage/deploy-minio-single-node-single-drive.html#id6> выясним команды установки, в особенности имя пакета.

2. На хосте "Objects" загрузим пакет (укажем выясненное имя пакета) и установим его:

    ```sh
    wget https://dl.min.io/server/minio/release/linux-amd64/archive/minio_20250228095516.0.0_amd64.deb -O minio.deb
    sudo dpkg -i minio.deb
    ```

3. Создадим пользователя и группу для службы:

    ```sh
    sudo groupadd -r minio-user
    sudo useradd -m -r -g minio-user minio-user
    ```

4. Создадим каталог для хранилища и установим созданного пользователя владельцем каталога:

    ```sh
    sudo mkdir /mnt/minio
    sudo chown minio-user:minio-user /mnt/minio
    ```

5. Создадим и откроем в редакторе файл настроек MinIO:

    ```sh
    sudo vim /etc/default/minio
    ```

6. В редакторе обеспечим следующее содержимое файла (вместо `password` зададим пароль пользователя для администратора MinIO):

    ```config
    MINIO_VOLUMES="/mnt/minio"
    MINIO_OPTS="--console-address :9001"
    MINIO_ROOT_USER=minioadmin
    MINIO_ROOT_PASSWORD=password
    ```

     Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

7. Включим в автозапуск и запустим службу MinIO:

    ```sh
    sudo systemctl enable minio
    sudo systemctl start minio
    ```

8. Откроем порты для службы MinIO:

    ```sh
    sudo ufw allow 9000/tcp
    sudo ufw allow 9001/tcp
    ```

9. На хосте "Client2" в веб-браузере откроем адрес <http://objects:9001/>.

    *Альтернатива: воспользоваться пробросом портов на хост внешней сети (`ssh -f -N -L 9001:objects:9001 user@192.168.122.201 -p 4224`) и в веб-браузере открыть адрес <http://localhost:9001/>*

10. Авторизуемся в MinIO.

### Настройка базового мониторинга

1. Откроем веб-интерфейс Zabbix сервера "Monitoring". Авторизуемся.

2. В веб-интерфейсе Zabbix слева в разделе "Data collection" выберем пункт "Host groups". В этом разделе выберем пункт "Hosts".

3. С помощью кнопки справа вверху "Create host" добавим хост для мониторинга: имя хоста - "objects"; отображаемое имя - "Object storage server"; шаблон - "ICMP ping" группы хостов - "Storage servers"; добавим интерфейс - агента с IP-адресом 10.0.1.13 и именем DNS "objects".

### Настройка TLS для веб-интерфейса объектного хранилища

1. На сервере "CA" перейдём в каталог удостоверяющего центра:

    ```sh
    cd /etc/ssl/ca
    ```

2. Создадим ключ и сертификат для веб-сервера `objects`:

    ```sh
    sudo openssl req -new -x509 -nodes -newkey rsa:4096 -days 3653 -utf8 -extensions v3_ca -out /etc/ssl/ca/certs/objects_https_cert.pem -keyout /etc/ssl/ca/private/objects_https_key.pem
    ```

    На запросы всех параметров, кроме параметров "Organizational Unit Name" и "Common Name" ответим нажатием Enter, т. к. эти параметры соответствуют параметрам, заданным в настройках OpenSSL по умолчанию; для параметра "Organizational Unit Name" зададим значение `Objects`; для параметра "Common Name" зададим значение `objects`:

    ```config
    Country Name (2 letter code) [RU]:
    State or Province Name (full name) [Tver region]:
    Locality Name (eg, city) [Tver]:
    Organization Name (eg, company) [Village]:
    Organizational Unit Name (eg, section) [CA]: Objects
    Common Name (e.g. server FQDN or YOUR name) [village]: objects
    Email Address [admin@village]:
    ```

    Сгенерированный сертификат находится в файле /etc/ssl/ca/cacert.pem и действует 10 лет (3653 дня).

3. Сгенерируем запрос на подпись сертификата:

    ```sh
    sudo openssl req -new -utf8 -key /etc/ssl/ca/private/objects_https_key.pem -out /etc/ssl/ca/requests/objects_https.csr
    ```

    На запросы всех параметров, кроме параметров "Organizational Unit Name" и "Common Name" ответим нажатием Enter, т. к. эти параметры соответствуют параметрам, заданным в настройках OpenSSL по умолчанию; для параметра "Organizational Unit Name" зададим значение `Objects`; для параметра "Common Name" зададим значение `objects`:

    ```config
    Country Name (2 letter code) [RU]:
    State or Province Name (full name) [Tver region]:
    Locality Name (eg, city) [Tver]:
    Organization Name (eg, company) [Village]:
    Organizational Unit Name (eg, section) [CA]: Objects
    Common Name (e.g. server FQDN or YOUR name) [village]: objects
    Email Address [admin@village]:

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

4. Создадим файл доменов для сертификата и откроем его в редекторе:

    ```sh
    sudo vim domains_objects.txt
    ```

5. В редакторе обеспечим следующие содержимое файла:

    ```config
    authorityKeyIdentifier=keyid,issuer
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = objects
    DNS.2 = objects.village
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

6. Подпишем сертификат:

    ```sh
    sudo openssl x509 -req -in requests/objects_https.csr -CA cacert.pem -CAkey private/cakey.pem -out newcerts/objects_https_cert.pem -days 3653 -CAcreateserial -extfile domains_objects.txt
    ```

7. Скопируем необходмые ключи и сертификаты в каталог пользователя для копирования его на серверы по `scp`, перейдём в этот каталог и установим пользователю права на файлы:

    ```sh
    sudo cp cacert.pem newcerts/objects_https_cert.pem private/objects_https_key.pem /home/user
    cd /home/user
    sudo chown user:user cacert.pem objects_https_cert.pem objects_https_key.pem
    ```

8. На сервере "Access" скопируем файлы ключей и сертификатов между каталогами пользователей сервера "CA" и сервера "Objects":

    ```sh
    scp ca:/home/user/{cacert.pem,objects_https_cert.pem,objects_https_key.pem} objects:/home/user
    ```

9. На сервере "CA" удалим файлы ключей и сертификатов из каталога пользователя:

    ```sh
    cd /home/user
    rm cacert.pem objects_https_cert.pem objects_https_key.pem
    ```

10. На хосте "Objects" авторизуемся как пользователь, от имени которого работает служба MinIO:

    ```sh
    sudo su - minio-user
    ```

11. Создадим каталоги для сертификатов и ключей:

    ```sh
    mkdir -p ~/.minio/certs/CAs
    ```

12. От имени основного пользователя, получая права суперпользователя, переместим ключи и сертификаты, задав им необходимые имена, и установим им владельца:

    ```sh
    sudo mv objects_https_cert.pem /home/minio-user/.minio/certs/public.crt
    sudo mv objects_https_key.pem /home/minio-user/.minio/certs/private.key
    sudo mv cacert.pem /home/minio-user/.minio/certs/CAs
    sudo chown -R minio-user:minio-user /home/minio-user/.minio/certs
    ```

13. Перезапустим службу MinIO:

    ```sh
    sudo systemctl restart minio
    ```

14. Проверим подключение по HTTPS: на хосте "Client 2" откроем в веб-браузере адрес <https://objects:9001/>. Поскольку сертификат "Village", подписавший сертификат сайта, ранее был добавлен в список доверенных корневых центров сертификации, страница должна открываться без предупреждений безопасности.
