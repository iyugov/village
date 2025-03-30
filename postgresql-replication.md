# Создание сервера репликации баз данных PostgreSQL

## Описание

Создаётся сервер репликации баз данных PostgreSQL на базе Ubuntu Server 24.04 с RAID 5. Сервер имеет адрес 10.0.1.9/24. Репликация осуществляется для базы данных `campus` с сервера "DB2".
Обеспечиваются базовые параметры безопасности.

## Требования

* [Создание сервера баз данных PostgreSQL](postgresql.md)
* [Создание коммутатора серверов баз данных](db-server-switch.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:

    * имя - "DB2R";
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

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet5` коммутатора "DBServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.
    * Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.
    * Выберем вариант установки "Ubuntu Server".
    * Для сетевого интерфейса укажем настройки для IPv4 вручную:
        * подсеть - 10.0.1.0/24;
        * адрес - 10.0.1.9;
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

14. Зададим отображаемое имя пользователя "User", имя сервера "db2r", имя пользователя "user"; зададим пароль пользователя.

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
    db2r    IN      A       10.0.1.9
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
    9       IN     PTR    db2r.village.
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

1. На сервере "DB2R" откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_db2r`.  
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на целевой сервер:

    ```sh
    ssh-copy-id -i ~/.ssh/id_db2r user@db2r
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_db2r user@db2r
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
    Host db2r
      Hostname db2r
      Port 22
      IdentityFile ~/.ssh/id_db2r
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер с учётом настроек:

    ```sh
    ssh db2r
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "DB2R" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

15. Настроим fail2ban:
    * Указания: [Настройка fail2ban для сервера SSH](fail2ban-ssh.md)

16. Настроим отправку локальных уведомлений о блокировках доступа:
    * Указания: [Настройка отправки уведомлений о блокировках доступа по SSH через локальный почтовый сервер](fail2ban-ssh-mail.md)
    * Параметры:
        * `HOSTNAME`: `db2r`

### Настройка сервера баз данных

1. На сервере "DB2R" установим сервер PostgreSQL:

    ```sh
    sudo apt install postgresql
    ```

2. Переключимся на пользователя `postgres`:

    ```sh
    sudo su - postgres
    ```

3. Создадим пользователя баз данных:

    ```sh
    createuser -P admin\
    exit
    ```

    Зададим пароль этого пользователя. Вернёмся к пользователю `user`.

4. Установим правило межсетевого экрана для PostgreSQL:

    ```sh
    sudo ufw allow postgresql
    ```

### Настройка репликации

#### Настройка основного сервера

1. На хосте "DB2" откроем файл конфигурации сервера PostgreSQL:

    ```sh
    sudo vim /etc/postgresql/16/main/postgresql.conf
    ```

    Добавим (раскомментируем, изменим) в файле следующие параметры:

    ```config
    listen_addresses = '*'
    logging_collector = on
    log_directory = 'pg_log'
    log_filename = 'postgresql.log'
    log_statement = 'all'
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 512MB
    synchronous_commit = on
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

2. Откроем файл настроек аутентификации по хостам:

    ```sh
    sudo vim /etc/postgresql/16/main/pg_hba.conf
    ```

    Добавим строку аутентификации:

    ```config
    host replication replicator db2r.village scram-sha-256
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

3. Откроем консоль сервера PostgreSQL:

    ```sh
    sudo -u postgres psql
    ```

4. В консоли создадим пользователя для репликации (вместо `password` зададим пароль этого пользователя):

    ```sql
    CREATE ROLE replicator WITH REPLICATION PASSWORD 'password' LOGIN;
    ```

    Выйдем из консоли:

    ```sql
    \q
    ```

5. Перезапустим сервер PostgreSQL:

    ```sh
    sudo systemctl restart postgresql
    ```

#### Настройка сервера репликации

1. На хосте "DB2R" остановим службу PostgreSQL:

    ```sh
    sudo systemctl stop postgresql
    ```

2. Удалим старые данные, если они есть:

    ```sh
    sudo -u postgres bash
    rm -rf /var/lib/postgresql/16/main/*
    exit
    ```

3. Создадим резервную копию баз данных с основного сервера:

    ```sh
    sudo -u postgres pg_basebackup -h db2 -D /var/lib/postgresql/16/main/ -P -U replicator --wal-method=stream

    ```

    Введём пароль пользователя `replicator` с основного сервера.

4. Откроем файл конфигурации сервера PostgreSQL:

    ```sh
    sudo vim /etc/postgresql/16/main/postgresql.conf
    ```

    Добавим (раскомментируем, изменим) в файле следующие параметры (вместо `PASSWORD` укажем пароль пользователя `replicator`):

    ```config
    primary_conninfo = 'host=db2 port=5432 user=replicator password=PASSWORD application_name=replica'
    recovery_target_timeline = 'latest'
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

5. Создадим файл флага для включения режима standby:

    ```sh
    sudo touch /var/lib/postgresql/16/main/standby.signal
    ```

6. Установим владельца каталога баз данных:

    ```sh
    sudo chown -R postgres:postgres /var/lib/postgresql/16/main
    ```

7. Запустим службу PostgreSQL:

    ```sh
    sudo systemctl start postgresql
    ```

#### Проверка репликации

1. На хосте "DB2" откроем консоль сервера PostgreSQL:

    ```sh
    sudo -u postgres psql
    ```

2. Выполним запрос о статусе репликации:

    ```sql
    SELECT * FROM pg_stat_replication;
    ```

    Должны быть выведены данные о репликации на хост "DB2R".
