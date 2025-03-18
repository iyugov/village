# Создание сервера мониторинга

## Описание

Создаётся сервер мониторинга на базе Ubuntu Server 24.04 с RAID 1 и Zabbix с использованием локальных серверов MySQL и Apache. Сервер имеет адрес 10.0.1.12/24.
Обеспечиваются базовые параметры безопасности.

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

1. На сервере "DB1" установим *fail2ban*:

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

1. На сервере "DB1" установим пакеты `postfix` и `mailutils`:

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
