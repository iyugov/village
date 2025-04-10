# Создание сервера доступа

## Описание

Создаётся сервер доступа на базе Ubuntu Server 24.04 с RAID 1. Сервер имеет адрес 10.0.1.2/24. Сервер имён для этого сервера имеет адрес 192.168.1.1 в сети базовой системы.
Обеспечиваются базовые параметры безопасности.

## Требования

* [Создание базовой сети](base-network.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:
    * имя - "DHCP";
    * ОС - Linux, Ubuntu (64-bit);
    * объём оперативной памяти - 2048 Мб, процессоры - 1;
    * накопители:
        * SATA, 2 Гб;
        * SATA, 5 Гб;
        * SATA, 5 Гб.
    * оптический привод для установочного образа;
    * сетевой адаптер: имеет тип Generic Driver с именем UDPTunnel.

2. Добавим виртуальную машину в GNS3 через шаблон.

3. Установим управление GNS3 для сетевых интерфейсов приложения и зададим число сетевых адаптеров - 1.

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к сетевому интерфейсу `Ethernet1` коммутатора "ServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.

7. Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.

8. Выберем вариант установки "Ubuntu Server".

9. Для сетевого интерфейса укажем настройки для IPv4 вручную:
    * подсеть - 10.0.1.0/24;
    * адрес - 10.0.1.2;
    * шлюз - 10.0.1.1;
    * сервер имён - 192.168.1.1 (в сети базовой системы).

10. Оставим настройку прокси пустой.

11. Для разметки накопителей выберем "Custom storage layout".

12. Создадим RAID 1:
    * выберем "Create software RAID (md)";
    * укажем имя "md0" и уровень 1 (зеркальный);
    * отметим устройства - 2 накопителя по 5 Гб.

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

22. Создадим и откроем файл настроек сервера SSH:

    ```sh
    sudo vim /etc/ssh/sshd_config.d/50-cloud-init.conf
    ```

23. Добавим в файл настройку неочевидного значения порта - для безопасности - 4224:

    ```config
    Port 4224
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).  
24. Перезапустим сервер SSH:

    ```sh
    sudo systemctl restart ssh
    ```

### Настройка перенаправления портов для сервера доступа

1. На роутере "Gateway" выполним перенаправление портов для SSH к серверу "Access" через роутер "ServerRouter":

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=4224 action=dst-nat to-addresses=10.0.0.1 to-ports=4224
    ```

2. На роутере "ServerRouter" выполним перенаправление портов для SSH к серверу "Access":

    ```mikrotik
    ip firewall nat add chain=dstnat in-interface=ether1 protocol=tcp dst-port=4224 action=dst-nat to-addresses=10.0.1.2 to-ports=22
    ```

3. На роутере "Gateway" выяснить его внешний IP-адрес (на интерфейсе `ether1`):

    ```mikrotik
    ip address print
    ```

4. С компьютера в сети базовой системы подключимся к серверу "Access" (изменим адрес 192.168.122.101 на выясненный адрес роутера "Gateway"):

    ```sh
    ssh user@192.168.122.101 -p 4224
    ```

    также проверим подключение к роутеру "Gateway":  

    ```sh
    telnet 192.168.1.101
    ```

    Подключение должно состояться.

5. В настройках хоста "Access" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

### Настройка безопасности сервера

#### Настройка доступа на сервер по ключу

1. На хосте, с которого доступен сервер "Access" и с которого собираемся подключаться по SSH к этому серверу, сгеренируем ключ (вместо `email` укажем адрес электронной почты администратора):

    ```sh
    ssh-keygen -t ed25519 -C "email"
    ```

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_village`.
    Зададим пустой пароль.

2. Выполним копирование сгенерированного ключа на сервер "Access" (укажем файл ключа и внешний адрес роутера "Gateway"):

    ```sh
    ssh-copy-id -i ~/.ssh/id_village -p 4224 user@192.168.122.201
    ```

3. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_village user@192.168.122.201 -p 4224
    ```

    Авторизация должна пройти без запроса пароля.

4. На сервере "Access" откроем файл настроек сервера SSH:

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

9. Настроим fail2ban:
    * Указания: [Настройка fail2ban для сервера SSH](fail2ban-ssh.md)

10. Настроим отправку локальных уведомлений о блокировках доступа:
    * Указания: [Настройка отправки уведомлений о блокировках доступа по SSH через локальный почтовый сервер](fail2ban-ssh-mail.md)
    * Параметры:
        * `HOSTNAME`: `access`
