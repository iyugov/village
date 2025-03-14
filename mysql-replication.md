# Создание сервера репликации баз данных MySQL

## Описание

Создаётся сервер репликации баз данных MySQL на базе Ubuntu Server 24.04 с RAID 5. Сервер имеет адрес 10.0.1.7/24. Репликация осуществляется для базы данных `campus` с сервера "DB1".
Обеспечиваются базовые параметры безопасности.

## Требования

* [Создание центра сертификации](ca.md)
* [Создание сервера баз данных MySQL](mysql.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:

    * имя - "DB1R";
    * ОС - Linux, Ubuntu (64-bit);
    * объём оперативной памяти - 2048 Мб, процессоры - 1;
    * накопители:
        * SATA, 2 Гб;
        * SATA, 20 Гб;
        * SATA, 20 Гб;
        * SATA, 20 Гб.
    * оптический привод для установочного образа;
    * сетевой адаптер: имеет тип Generic Driver с именем UDPTunnel.

2. Добавим виртуальную машину в GNS3 через шаблон.

3. Установим управление GNS3 для сетевых интерфейсов приложения и зададим число сетевых адаптеров - 1.

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet6` коммутатора "ServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.
    * Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.
    * Выберем вариант установки "Ubuntu Server".
    * Для сетевого интерфейса укажем настройки для IPv4 вручную:
        * подсеть - 10.0.1.0/24;
        * адрес - 10.0.1.7;
        * шлюз - 10.0.1.1;
        * сервер имён - 10.0.1.3;
        * домен поиска - "village".

7. Оставим настройку прокси пустой.

8. Для разметки накопителей выберем "Custom storage layout".

9. Создадим RAID 1:
    * выберем "Create software RAID (md)";
    * укажем имя "md0" и уровень 5;
    * отметим устройства - 3 накопителя по 20 Гб.

10. Создадим LVM:
    * выберем "Create volume group (LVM)";
    * укажем имя "vg0" и расположение - "md0".

11. Сделаем накопитель размером 2 Гб загрузочным ("Use As Boot Device"). В свободном пространстве этого накопителя создадим раздел GPT ("Add GPT Partition") с размером по умолчанию, файловой системой ext4 и точкой монтирования "/boot".

12. В свободном пространстве группы логических томов "vg0" создадим логический том ("Create Logical Volume") с именем "lv-0", размером по умолчанию, файловой системой ext4 и точкой монтирования "/".

13. Проверим разметку, завершим её, согласимся на изменения.

14. Зададим отображаемое имя пользователя "User", имя сервера "db1r", имя пользователя "user"; зададим пароль пользователя.

15. Пропустим настройку Ubuntu Pro.

16. Отметим установку сервера OpenSSH.

17. Дождёмся окончания установки. Перезагрузим сервер. Авторизуемся.

18. Обновим пакеты:

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
    db1r    IN      A       10.0.1.7
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
    7       IN     PTR    db1r.village.
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

1. На сервере имён откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_db1r`.  
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на сервер баз данных:

    ```sh
    ssh-copy-id -i ~/.ssh/id_db1r user@db1r
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_db1r user@db1r
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
    Host db1r
      Hostname db1r
      Port 22
      IdentityFile ~/.ssh/id_db1r
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер с учётом настроек:

    ```sh
    ssh db1r
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "DB1R" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

#### Настройка fail2ban на сервере

1. На сервере "DB1R" установим *fail2ban*:

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

1. На сервере "DB1R" установим пакеты `postfix` и `mailutils`:

    ```sh
    sudo apt update
    sudo apt install postfix mailutils
    ```

2. При установке в окне настройки Postfix выберем "Local only", укажем иия системы - `db1r`.

3. Проверим состояние службы Postfix:

    ```sh
    sudo systemctl status postfix
    ```

4. Выполним отправку тестового сообщения:

    ```sh
    echo "Тестовое сообщение" | mail -s "Тестовая тема" user@db1r
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
    destemail = user@db1r
    sender = fail2ban@db1r
    mta = mail
    action = %(action_mwl)s
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

9. Перезапустим службу *fail2ban*:

    ```sh
    sudo systemctl restart fail2ban
    ```

10. Проверим отправку сообщений при блокировке. Также уведомления будут отправляться о запуске и завершении работы *fail2ban*.

### Настройка сервера баз данных

1. На сервере "DB1R" установим сервер MySQL:

    ```sh
    sudo apt install mysql-server
    ```

2. Запустим консоль сервера MySQL:

    ```sh
    sudo mysql
    ```

3. Создадим пользователя баз данных (вместо `password` зададим пароль этого пользователя):

    ```sql
    CREATE USER 'admin'@'localhost' IDENTIFIED BY 'password';
    ```

4. Дадим созданному пользователю полные права:

    ```sql
    GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
    ```

5. Выйдем из консоли:

    ```sql
    EXIT;
    ```

### Настройка репликации

#### Создание ключей и сертификатов

1. На сервере "CA" перейдём в каталог удостоверяющего центра:

    ```sh
    cd /etc/ssl/ca
    ```

2. Создадим ключ и сертификат для сервера MySQL `db1`:

    ```sh
    sudo openssl genpkey -algorithm ed25519 -out private/db1_mysql_key.pem
    sudo openssl req -new -key private/db1_mysql_key.pem -out requests/db1_mysql.csr -subj "/CN=db1"
    sudo openssl x509 -req -in requests/db1_mysql.csr -CA ca_cert.pem -CAkey private/ca_key.pem -out newcerts/db1_mysql_cert.pem -days 3653 -CAcreateserial
    ```

3. Создадим ключ и сертификат для сервера MySQL `db1r`:

    ```sh
    sudo openssl genpkey -algorithm ed25519 -out private/db1r_mysql_key.pem
    sudo openssl req -new -key private/db1r_mysql_key.pem -out requests/db1r_mysql.csr -subj "/CN=db1r"
    sudo openssl x509 -req -in requests/db1r_mysql.csr -CA ca_cert.pem -CAkey private/ca_key.pem -out newcerts/db1r_mysql_cert.pem -days 3653 -CAcreateserial
    ```

4. Скопируем необходмые ключи и сертификаты в каталог пользователя для копирования его на серверы по `scp`, перейдём в этот каталог и установим пользователю права на файлы:

    ```sh
    sudo cp ca_cert.pem newcerts/db1_mysql_cert.pem newcerts/db1r_mysql_cert.pem private/db1_mysql_key.pem private/db1r_mysql_key.pem /home/user
    cd /home/user
    sudo chown user:user ca_cert.pem db1_mysql_cert.pem db1r_mysql_cert.pem db1_mysql_key.pem db1r_mysql_key.pem
    ```

5. На сервере "Access" скопируем файлы ключей и сертификатов  между каталогами пользователей сервера "CA" и сервера "DB1":

    ```sh
    scp ca:/home/user/{ca_cert.pem,db1_mysql_cert.pem,db1r_mysql_cert.pem,db1_mysql_key.pem,db1r_mysql_key.pem} db1:/home/user
    ```

6. На сервере "Access" скопируем файлы ключей и сертификатов между каталогами пользователей сервера "CA" и сервера "DB1R":

    ```sh
    scp ca:/home/user/{ca_cert.pem,db1_mysql_cert.pem,db1r_mysql_cert.pem,db1_mysql_key.pem,db1r_mysql_key.pem} db1r:/home/user
    ```

7. На сервере "CA" удалим файлы ключей и сертификатов  из каталога пользователя:

    ```sh
    cd /home/user
    rm ca_cert.pem db1_mysql_cert.pem db1r_mysql_cert.pem db1_mysql_key.pem db1r_mysql_key.pem
    ```

8. На сервере "DB1" создадим каталог для сертификатов и ключей, скопируем файлы в него, установим права:

    ```sh
    cd /home/user
    sudo mkdir /etc/mysql/ssl
    sudo mv ca_cert.pem db1_mysql_cert.pem db1r_mysql_cert.pem db1_mysql_key.pem db1r_mysql_key.pem /etc/mysql/ssl
    sudo chown -R mysql:mysql /etc/mysql/ssl
    sudo chmod -R 600 /etc/mysql/ssl
    sudo chmod 700 /etc/mysql/ssl
    ```

9. На сервере "DB1R" создадим каталог для сертификатов и ключей, скопируем файлы в него, установим права:

    ```sh
    cd /home/user
    sudo mkdir /etc/mysql/ssl
    sudo mv ca_cert.pem db1_mysql_cert.pem db1r_mysql_cert.pem db1_mysql_key.pem db1r_mysql_key.pem /etc/mysql/ssl
    sudo chown -R mysql:mysql /etc/mysql/ssl
    sudo chmod -R 600 /etc/mysql/ssl
    sudo chmod 700 /etc/mysql/ssl
    ```

#### Настройка основного сервера

1. На сервере "DB1" откроем файл конфигурации сервера MySQL:

    ```sh
    sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
    ```

2. В разделе `[mysql]` установим, при необходимости раскомментировав или дописав, значения параметров:

    ```config
    bind-address = 0.0.0.0
    server-id = 1
    log_bin = /var/log/mysql/mysql-bin.log
    binlog_do_db = campus
    ssl_ca = /etc/mysql/ssl/ca_cert.pem
    ssl_cert = /etc/mysql/ssl/db1_mysql_cert.pem
    ssl_key = /etc/mysql/ssl/db1_mysql_key.pem
    require_secure_transport=ON
    ```

3. Перезапустим сервер MySQL:

    ```sh
    sudo systemctl restart mysql
    ```

4. Запустим консоль сервера MySQL:

    ```sh
    sudo mysql
    ```

5. Создадим пользователя для репликации (вместо `password` зададим пароль этого пользователя):

    ```sql
    CREATE USER 'replicator'@'%' IDENTIFIED BY 'password' REQUIRE SSL;
    GRANT REPLICATION SLAVE ON *.* TO 'replicator'@'%';
    ```

6. Заблокируем таблицы баз данных:

    ```sql
    FLUSH TABLES WITH READ LOCK;
    SHOW MASTER STATUS;
    ```

    Запомним (запишем) имя файла (поле `File`) и позицию (поле `Position`) из выведенной таблицы.

7. Разблокируем таблицы и выйдем из консоли сервера:

    ```sql
    UNLOCK TABLES;
    EXIT;
    ```

8. Создадим дамп базы данных `campus`:

    ```sh
    mysqldump -u admin -p --databases campus > /home/user.dump.sql
    ```

    Введём пароль пользователя `admin`.

9. На сервере "Access" выполним копирование дампа с "DB1" на "DB1R":

    ```sh
    scp db1:/home/user/dump.sql db1r:/home/user
    ```

10. На сервере "DB1" удалим дамп:

    ```sh
    rm /home/user/dump.sql
    ```

11. На сервере "DB1R" выполним восстановление базы данных из дампа и удалим дамп:

    ```sh
    sudo mysql < /home/user/dump.sql && rm /home/user/dump.sql
    ```

12. На сервере "DB1R" откроем файл конфигурации сервера MySQL:

    ```sh
    sudo vim /etc/mysql/mysql.conf.d/mysqld.cnf
    ```

13. В разделе `[mysql]` установим, при необходимости раскомментировав или дописав, значения параметров:

    ```config
    server-id = 2
    relay-log = /var/log/mysql/relay-bin.log
    ssl_ca = /etc/mysql/ssl/ca_cert.pem
    ssl_cert = /etc/mysql/ssl/db1_mysql_cert.pem
    ssl_key = /etc/mysql/ssl/db1_mysql_key.pem
    require_secure_transport=ON
    ```

14. Перезапустим сервер MySQL:

    ```sh
    sudo systemctl restart mysql
    ```

15. Запустим консоль сервера MySQL:

    ```sh
    sudo mysql
    ```

16. Выполним настройку репликации, заменив значения параметров `MASTER_LOG_FILE` и `MASTER_LOG_POS` ранее записанными значениями имени файла и позиции, а значение параметра `MASTER_PASSWORD` - паролем пользователя `replicator` на сервера MySQL `db1`:

    ```sql
    CHANGE MASTER TO
        MASTER_HOST='db1',
        MASTER_USER='replicator',
        MASTER_PASSWORD='password',
        MASTER_LOG_FILE='mysql-bin.000001',
        MASTER_LOG_POS=715,
        MASTER_SSL=1,
        MASTER_SSL_CA='/etc/mysql/ssl/ca_cert.pem',
        MASTER_SSL_CERT='/etc/mysql/ssl/db1r_mysql_cert.pem',
        MASTER_SSL_KEY='/etc/mysql/ssl/db1r_mysql_key.pem';
    ```

17. Запустим репликацию:

    ```sql
    START SLAVE;
    ```

18. Проверим статус репликации:

    ```sql
    SHOW SLAVE STATUS\G;
    ```

    Значения параметров `Slave_IO_Running` и `Slave_SQL_Running` должны быть `Yes`.
