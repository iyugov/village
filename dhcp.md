# Создание DHCP-сервера

## Описание

Создаётся DHCP-сервер на базе Ubuntu Server 24.04 с RAID 1. Сервер имеет адрес 10.0.1.2/24 и управляет адресами в подсети 172.16.0.0/16. Сервер имён для этого сервера имеет адрес 192.168.1.1 в сети базовой системы.

## Требования

* [Создание роутера для сегмента серверов](server-router.md)

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

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к сетевому интерфейсу `ether2` роутера "ServerRouter".

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

    Определим имя сетевого интерфейса, подключённого к роутеру "ServerRouter", т. е. имеющего IP-адрес 10.0.1.2.

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
