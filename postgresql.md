# Создание сервера баз данных PostgreSQL

## Описание

Создаётся сервер баз данных PostgreSQL на базе Ubuntu Server 24.04 с RAID 5. Сервер имеет адрес 10.0.1.8/24.
Обеспечиваются базовые параметры безопасности.

## Требования

* [Создание сервера имён](ns.md)

## Выполнение

### Создание хоста

1. Создадим виртуальную машину VirtualBox со следующими параметрами:

    * имя - "DB2";
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

4. Подключим сетевой интерфейс `Ethernet0` виртуальной машины к порту `Ethernet7` коммутатора "ServerSwitch".

5. Загрузим виртуальную машину с установочного образа Ubuntu Server 24.04.1 LTS.

6. В меню загрузки выберем "Try or Install Ubuntu Server" и дождёмся окончания загрузки.
    * Выберем язык "English". Выберем раскладку клавиатуры "Russian" и вариант "Russian". Определим клавишу "Caps Lock" для переключения раскладок.
    * Выберем вариант установки "Ubuntu Server".
    * Для сетевого интерфейса укажем настройки для IPv4 вручную:
        * подсеть - 10.0.1.0/24;
        * адрес - 10.0.1.8;
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

14. Зададим отображаемое имя пользователя "User", имя сервера "db2", имя пользователя "user"; зададим пароль пользователя.

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
    db2     IN      A       10.0.1.8
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
    8       IN     PTR    db2.village.
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

1. На сервере "DB2" откроем файл настроек сервера SSH:

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

    Укажем путь и имя файла для сохранения ключа, например, `/home/user/.ssh/id_db2`.  
    Зададим пустой пароль.

7. Выполним копирование сгенерированного ключа на целевой сервер:

    ```sh
    ssh-copy-id -i ~/.ssh/id_db2 user@db2
    ```

8. Выполним вход по ключу:

    ```sh
    ssh -i ~/.ssh/id_db2 user@db2
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
    Host db2
      Hostname db2
      Port 22
      IdentityFile ~/.ssh/id_db2
      IdentitiesOnly yes
    ```

    Сохраним изменения, выйдем из редактора (`Esc`, `Shift`+`z`, `Shift`+`z`).

13. Выполним вход на сервер с учётом настроек:

    ```sh
    ssh db2
    ```

    Авторизация должна пройти автоматически.

14. В настройках хоста "DB2" в GNS3 установим флажок "Start VM in headless mode", остановим и запустим хост.

15. Настроим fail2ban:
    * Указания: [Настройка fail2ban для сервера SSH](fail2ban-ssh.md)

16. Настроим отправку локальных уведомлений о блокировках доступа:
    * Указания: [Настройка отправки уведомлений о блокировках доступа по SSH через локальный почтовый сервер](fail2ban-ssh-mail.md)
    * Параметры:
        * `HOSTNAME`: `db2`

### Настройка сервера баз данных

1. На сервере "DB2" установим сервер PostgreSQL:

    ```sh
    sudo apt install postgresql
    ```

2. Переключимся на пользователя `postgres`:

    ```sh
    sudo su - postgres
    ```

3. Создадим пользователя баз данных:

    ```sh
    createuser -P admin
    exit
    ```

    Зададим пароль этого пользователя. Вернёмся к пользователю `user`.

4. Установим правило межсетевого экрана для PostgreSQL:

    ```sh
    sudo ufw allow postgresql
    ```

### Создание базы данных на сервере

1. На сервере "DB2" запустим консоль сервера PostgreSQL:

    ```sh
    sudo -u postgres psql
    ```

2. Создадим базу данных `campus`:

    ```sql
    CREATE DATABASE campus;
    ```

3. Сделаем созданную базу данных текущей:

    ```sql
    \c campus;
    ```

4. Создадим таблицу `students`:

    ```sql
    CREATE TABLE students (
        id SERIAL PRIMARY KEY,
        first_name VARCHAR(50) NOT NULL,
        last_name VARCHAR(50) NOT NULL,
        birth_date DATE,
        email VARCHAR(100) UNIQUE NOT NULL
    );
    ```

5. Создадим таблицу `courses`:

    ```sql
    CREATE TABLE courses (
        id SERIAL PRIMARY KEY,
        course_title VARCHAR(100) NOT NULL,
        credits INT,
        description TEXT
    );
    ```

6. Создадим таблицу `enrollments`:

    ```sql
    CREATE TABLE enrollments (
        id SERIAL PRIMARY KEY,
        student_id INT NOT NULL,
        course_id INT NOT NULL,
        enrollment_date DATE,
        grade CHAR(1),
        CONSTRAINT fk_student
            FOREIGN KEY (student_id) 
            REFERENCES students(id)
            ON DELETE CASCADE,
        CONSTRAINT fk_course
            FOREIGN KEY (course_id) 
            REFERENCES courses(id)
            ON DELETE CASCADE
    );
    ```

7. Внесём в таблицы начальные данные:

    ```sql
    INSERT INTO students (first_name, last_name, birth_date, email) VALUES
    ('Иван', 'Иванов', '1990-05-15', 'ivanov@example.com'),
    ('Мария', 'Петрова', '1992-08-22', 'petrova@example.com'),
    ('Пётр', 'Сидоров', '1988-12-03', 'sidorov@example.com');
    
    INSERT INTO courses (course_title, credits, description) VALUES
    ('Математика', 5, 'Основы алгебры и геометрии'),
    ('История', 3, 'Изучение истории древнего мира'),
    ('Компьютерные науки', 4, 'Введение в программирование и алгоритмы');
    
    INSERT INTO enrollments (student_id, course_id, enrollment_date, grade) VALUES
    (1, 1, '2023-09-01', 'A'),
    (2, 2, '2023-09-02', 'B'),
    (3, 3, '2023-09-03', 'A'),
    (1, 3, '2023-09-04', 'B'),
    (2, 1, '2023-09-05', 'C');
    ```

8. Выйдем из консоли:

    ```sql
    \q
    ```
