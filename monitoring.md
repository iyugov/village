# Создание сервера мониторинга

## Описание

Создаётся сервер мониторинга на базе Ubuntu Server 24.04 с RAID 1 и Zabbix с использованием локальных серверов MySQL и Apache. Сервер имеет адрес 10.0.1.12/24.
Обеспечиваются базовые параметры безопасности. Настраивается базовый мониторинг имеющихся серверов по ICMP ping.

## Требования

* [Создание коммутаторов сегментов серверов](server-segments-switches.md)
* [Создание хоста графического клиента](client-gui.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:
    * имя - "Monitoring";
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

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet3` коммутатора "AdminServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.

7. Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.

8. Выберем вариант установки "Ubuntu Server".

9. Для сетевого интерфейса укажем настройки для IPv4 вручную:
    * подсеть - 10.0.1.0/24;
    * адрес - 10.0.1.12;
    * шлюз - 10.0.1.1;
    * сервер имён - 10.0.1.3;
    * домен поиска - "village".

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

17. Зададим отображаемое имя пользователя "User", имя сервера "monitoring", имя пользователя "user"; зададим пароль пользователя.

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
    monitoring IN      A       10.0.1.12
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
    12      IN     PTR    monitoring.village.
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

1. На сервере "Monitoring" откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_monitoring`.  
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на целевой сервер:

    ```sh
    ssh-copy-id -i ~/.ssh/id_monitoring user@monitoring
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_monitoring user@monitoring
    ```

    Авторизация должна пройти без запроса пароля.

9. На сервере имён откроем файл настроек сервера SSH:

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

12. На сервере "Access" откроем файл настроек подключений по SSH:

    ```sh
    vim ~/.ssh/config
    ```

    Добавим в файл следующие строки:  

    ```config
    Host monitoring
      Hostname monitoring
      Port 22
      IdentityFile ~/.ssh/id_monitoring
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер с учётом настроек:

    ```sh
    ssh monitoring
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "Monitoring" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

#### Настройка fail2ban на сервере

1. На сервере "Monitoring" установим *fail2ban*:

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

1. На сервере "Monitoring" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `monitoring`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@monitoring
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
    destemail = user@monitoring
    sender = fail2ban@monitoring
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим отправку сообщений при блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.

### Установка Zabbix

*На сайте <https://www.zabbix.com/download> можно выбрать параметры установки и получить команды для неё.*

*Описываемым параметрам установки соответствует следующая ссылка:
<https://www.zabbix.com/download?zabbix=7.2&os_distribution=ubuntu&os_version=24.04&components=server_frontend_agent&db=mysql&ws=apache>*

1. На сервере "Monitoring" загрузим пакет с источниками пакетов Zabbix, установим источники, удалим файл пакета, а затем установим сами компоненты Zabbix, а также сервер MySQL:

    ```sh
    sudo wget https://repo.zabbix.com/zabbix/7.2/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_7.2+ubuntu24.04_all.deb
    sudo dpkg -i zabbix-release_latest_7.2+ubuntu24.04_all.deb
    rm zabbix-release_latest_7.2+ubuntu24.04_all.deb
    sudo apt update
    sudo apt install zabbix-server-mysql zabbix-frontend-php zabbix-apache-conf zabbix-sql-scripts zabbix-agent mysql-server
    ```

2. Откроем консоль сервера MySQL:

    ```sh
    sudo mysql
    ```

3. Создадим базу данных `zabbix`, пользователя `zabbix` для неё (вместо `password` зададим пароль пользователя), установим флаг доверия создаваемым функциям (для импорта конфигурации) и выйдем из консоли:

    ```sql
    CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
    CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'password';
    GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
    SET GLOBAL log_bin_trust_function_creators = 1;
    EXIT;
    ```

4. Импортируем конфигурацию базы данных для Zabbix:

    ```sh
    zcat /usr/share/zabbix/sql-scripts/mysql/server.sql.gz | mysql --default-character-set=utf8mb4 -uzabbix -p zabbix
    ```

    Введём пароль для пользователя `zabbix` базы данных.
    Дождёмся окончания импорта.

5. Откроем консоль сервера MySQL:

    ```sh
    sudo mysql
    ```

6. Снимем флаг доверия создаваемым функциям (для импорта конфигурации) и выйдем из консоли:

    ```sql
    EXIT;
    ```

7. Откроем файл конфигурации сервера Zabbix:

    ```sh
    sudo vim /etc/zabbix/zabbix_server.conf
    ```

8. В файле раскомментируем строку с параметром `DBPassword` или допишем новую такую строку и в качестве значения параметра укажем пароль для пользователя `zabbix` базы данных (вместо `password`):

    ```config
    DBPassword=password
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

9. Перезапустим и включим в автоматическую загрузку служб Zabbix и веб-сервера:

    ```sh
    sudo systemctl restart zabbix-server zabbix-agent apache2
    sudo systemctl enable zabbix-server zabbix-agent apache2
    ```

10. Установим правило межсетевого экрана для веб-сервера (HTTP):

    ```sh
    sudo ufw allow http
    ```

11. На хосте "Client2" в веб-браузере откроем адрес <http://monitoring/zabbix>.

    *Альтернатива: воспользоваться пробросом портов на хост внешней сети (`ssh -f -N -L 8080:monitoring:80 user@192.168.122.201 -p 4224`) и в веб-браузере открыть адрес <http://localhost:8080/zabbix>*

12. В веб-браузере выберем язык: "English (en_US)". Нажмём кнопку "Next step".

13. Проверим предварительные требования (везде должны стоять отметки "OK"). Нажмём кнопку "Next step".

14. Укажем параметры подключения:
    * тип базы данных - MySQL;
    * хост базы данных - `localhost`;
    * порт базы данных - 0 (по умолчанию);
    * имя базы данных - `zabbix`;
    * хранение учётных данных - в простоим текстовом формате;
    * имя пользователя - `zabbix`;
    * пароль - тот пароль, который был установлен для этого пользователя базы данных.
    Нажмём кнопку "Next step".

15. Зададим имя сервера Zabbix - "Monitoring". Выберем часовой пояс - "(UTC+03:00) Europe/Moscow". Выберем тему оформления по умолчанию - "Blue". Нажмём кнопку "Next step".

16. Проверим ранее указанные параметры настройки. При необходимости - вернёмся назад (кнопки "Back") и скорректируем настройки. Нажмём кнопку "Next step".

17. Дождёмся окончания установки. Нажмём кнопку "Finish".

18. Авторизуемся в Zabbix с помощью имени пользователя "Admin" и пароля "zabbix".

19. Слева в разделе "User settings" выберем пункт "Profile". Нажмём кнопку "Change password". Введём прежний пароль и зададим новый пароль, указав его дважды. Нажмём кнопку "Upgrade". Согласимся на завершение сеанса.

20. Авторизуемся в Zabbix с именем пользователя "Admin" и установленным паролем.

### Настройка веб-сервера для мониторинга

#### Настройка перенаправления

1. В консоли хоста "Monitoring" откроем файл конфигурации сайта веб-сервера:

    ```sh
    sudo vim /etc/apache2/sites-enabled/000-default.conf
    ```

2. В разделе `<VirtualHost *:80>` добавим настройку перенаправления на каталог "Zabbix":

    ```config
    RedirectMatch 301 ^/$ /zabbix/
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. Перезапустим службу веб-сервера:

    ```sh
    sudo systemctl restart apache2
    ```

#### Создание ключей и сертификатов

1. На сервере "CA" перейдём в каталог удостоверяющего центра:

    ```sh
    cd /etc/ssl/ca
    ```

2. Создадим ключ и сертификат для веб-сервера `monitoring`:

    ```sh
    sudo openssl req -new -x509 -nodes -newkey rsa:4096 -days 3653 -utf8 -extensions v3_ca -out /etc/ssl/ca/certs/monitoring_https_cert.pem -keyout /etc/ssl/ca/private/monitoring_https_key.pem
    ```

    На запросы всех параметров, кроме параметров "Organizational Unit Name" и "Common Name" ответим нажатием Enter, т. к. эти параметры соответствуют параметрам, заданным в настройках OpenSSL по умолчанию; для параметра "Organizational Unit Name" зададим значение `Monitoring`; для параметра "Common Name" зададим значение `monitoring`:

    ```config
    Country Name (2 letter code) [RU]:
    State or Province Name (full name) [Tver region]:
    Locality Name (eg, city) [Tver]:
    Organization Name (eg, company) [Village]:
    Organizational Unit Name (eg, section) [CA]: Monitoring
    Common Name (e.g. server FQDN or YOUR name) [village]: monitoring
    Email Address [admin@village]:
    ```

    Сгенерированный сертификат находится в файле /etc/ssl/ca/cacert.pem и действует 10 лет (3653 дня).

3. Сгенерируем запрос на подпись сертификата:

    ```sh
    sudo openssl req -new -utf8 -key /etc/ssl/ca/private/monitoring_https_key.pem -out /etc/ssl/ca/requests/monitoring_https.csr
    ```

    На запросы всех параметров, кроме параметров "Organizational Unit Name" и "Common Name" ответим нажатием Enter, т. к. эти параметры соответствуют параметрам, заданным в настройках OpenSSL по умолчанию; для параметра "Organizational Unit Name" зададим значение `Monitoring`; для параметра "Common Name" зададим значение `monitoring`:

    ```config
    Country Name (2 letter code) [RU]:
    State or Province Name (full name) [Tver region]:
    Locality Name (eg, city) [Tver]:
    Organization Name (eg, company) [Village]:
    Organizational Unit Name (eg, section) [CA]: Monitoring
    Common Name (e.g. server FQDN or YOUR name) [village]: monitoring
    Email Address [admin@village]:

    Please enter the following 'extra' attributes
    to be sent with your certificate request
    A challenge password []:
    An optional company name []:
    ```

4. Создадим файл доменов для сертификата и откроем его в редекторе:

    ```sh
    sudo vim domains_monitoring.txt
    ```

5. В редакторе обеспечим следующие содержимое файла:

    ```config
    authorityKeyIdentifier=keyid,issuer
    keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
    subjectAltName = @alt_names
    [alt_names]
    DNS.1 = monitoring
    DNS.2 = monitoring.village
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

6. Подпишем сертификат:

    ```sh
    sudo openssl x509 -req -in requests/monitoring_https.csr -CA cacert.pem -CAkey private/cakey.pem -out newcerts/monitoring_https_cert.pem -days 3653 -CAcreateserial -extfile domains_monitoring.txt
    ```

7. Скопируем необходмые ключи и сертификаты в каталог пользователя для копирования его на серверы по `scp`, перейдём в этот каталог и установим пользователю права на файлы:

    ```sh
    sudo cp cacert.pem newcerts/monitoring_https_cert.pem private/monitoring_https_key.pem /home/user
    cd /home/user
    sudo chown user:user cacert.pem monitoring_https_cert.pem monitoring_https_key.pem
    ```

8. На сервере "Access" скопируем файлы ключей и сертификатов между каталогами пользователей сервера "CA" и сервера "Monitoring":

    ```sh
    scp ca:/home/user/{cacert.pem,monitoring_https_cert.pem,monitoring_https_key.pem} monitoring:/home/user
    ```

9. На сервере "CA" удалим файлы ключей и сертификатов из каталога пользователя:

    ```sh
    cd /home/user
    rm cacert.pem monitoring_https_cert.pem monitoring_https_key.pem
    ```

10. На сервере "Monitoring" создадим каталог для сертификатов и ключей, скопируем файлы в него, установим права:

    ```sh
    cd /home/user
    sudo mkdir /etc/apache2/ssl
    sudo mv cacert.pem monitoring_https_cert.pem monitoring_https_key.pem /etc/apache2/ssl
    sudo chown -R www-data:www-data /etc/apache2/ssl
    sudo chmod -R 600 /etc/apache2/ssl
    sudo chmod 700 /etc/apache2/ssl
    ```

11. Откроем файл конфигурации сайта веб-сервера:

    ```sh
    sudo vim /etc/apache2/sites-enabled/000-default.conf
    ```

12. Добавим в файл раздел для HTTPS:

    ```config
    <VirtualHost *:443>
        ServerName monitoring.village
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html
        RedirectMatch 301 ^/$ /zabbix/
        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined
        SSLEngine on
        SSLCertificateFile /etc/apache2/ssl/monitoring_https_cert.pem
        SSLCACertificateFile /etc/apache2/ssl/cacert.pem
        SSLCertificateKeyFile /etc/apache2/ssl/monitoring_https_key.pem
    </VirtualHost>
    ```

    Удалим раздел ``.

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Включим модуль веб-сервера для SSL/TLS:

    ```sh
    sudo a2enmod ssl
    ```

14. Перезапустим службу веб-сервера:

    ```sh
    sudo systemctl restart apache2
    ```

15. Установим правило межсетевого экрана для веб-сервера (HTTPS):

    ```sh
    sudo ufw allow https
    ```

16. Проверим подключение по HTTPS: на хосте "Client 2" откроем в веб-браузере адрес <https://monitoring/>, отобразим сертификат сайта и сертификат подписавшего его удостоверяющего центра, добавим сертификат удостоверяющего центра в список доверенных центров сертификации, откроем адрес повторно. Должна отобразиться страница по адресу <https://monitoring/zabbix/> без предупреждений безопасности.

### Настройка базового мониторинга

1. На хосте "Monitoring" откроем файл конфигурации сервера Zabbix:

    ```sh
    sudo vim /etc/zabbix/zabbix_server.conf
    ```

2. В файле раскомментируем или допишем строки с параметрами программ "ping" и установим им следующие значения:

    ```config
    FpingLocation=/usr/bin/fping
    Fping6Location=/usr/bin/fping6
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. Перезапустим сервер и агент Zabbix:

    ```sh
    sudo systemctl restart zabbix-server zabbix-agent
    ```

4. В веб-интерфейсе Zabbix слева в разделе "Data collection" выберем пункт "Host groups".

5. С помощью кнопки справа вверху "Create host group" добавим группы "Admin servers", "Base servers", "Database servers", "Storage servers".

6. В разделе "Data collection" выберем пункт "Templates".

7. В списке шаблонов найдём "ICMP ping", нажмём на "Items" для него. Для каждого из трёх элементов, открыв его, добавим в ключ ("Key") `[{HOST.IP}]`; таким образом, ключи должны быть такие:
    * `icmpping[{HOST.IP}]`
    * `icmppingloss[{HOST.IP}]`
    * `icmppingsec[{HOST.IP}]`

8. В разделе "Data collection" выберем пункт "Hosts".

9. С помощью кнопки справа вверху "Create host" добавим следующие хосты для мониторинга:

    * имя хоста: "access"; отображаемое имя: "Access server"; группы хостов: "Admin servers";
    * имя хоста: "ca"; отображаемое имя: "Certificate authority server"; группы хостов: "Admin servers";
    * имя хоста: "dhcp"; отображаемое имя: "DHCP server"; группы хостов: "Base servers";
    * имя хоста: "ns"; отображаемое имя: "Name server"; группы хостов: "Base servers";
    * имя хоста: "db1"; отображаемое имя: "MySQL server"; группы хостов: "Database servers";
    * имя хоста: "db1r"; отображаемое имя: "MySQL replica server"; группы хостов: "Database servers";
    * имя хоста: "db2"; отображаемое имя: "PostgreSQL server"; группы хостов: "Database servers";
    * имя хоста: "db2r"; отображаемое имя: "PostgreSQL replica server"; группы хостов: "Database servers";
    * имя хоста: "smb"; отображаемое имя: "Samba server"; группы хостов: "Storage servers";
    * имя хоста: "nfs"; отображаемое имя: "NFS server"; группы хостов: "Storage servers".

10. Для каждого из добавленных хостов добавим шаблон "ICMP ping": откроем свойства хоста, начнём вводить "ICMP ping", подтвердим выбор; также добавим интерфейс - агента, укажем IP-адрес и имя соответствующего хоста.

11. Подождём около минуты, затем выберем раздел "Dashboards". При недоступных серверах должны отображаться соответствующие сообщения.
