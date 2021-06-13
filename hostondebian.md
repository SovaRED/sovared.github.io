# Послідовність для налаштування хостінгу на Debian 10  та встановлення Wordpress
## Підготовчі роботи перед встановленням LAMP
Після встановлення системи оновити всі компоненти.

    apt update
    apt upgrade
    apr full-upgrade
    apt autoremove

Тепер треба встановити Grub на другий диск

    dpkg-reconfigure grub-pc
  
Встановити мінімальний набір необхідного софта

    apt install mc htop iftop curl sudo links
   
Вибрати редактор за замовченням
 
    select-editor
    
Включити підсвічування синтаксису всіх файлів, які не позначені явно в файлі `/usr/share/mc/syntax/Syntax` синтаксисом для `sh` і `bash` скриптів. Цей універсальний синтаксис нормально підходить для конфігураційних файлів, з якими найчастіше доводиться працювати на сервері. Перезаписуємо файл `unknown.syntax`. Саме цей шаблон буде застосовуватися до `.conf` і `.cf` файлів, так як до них явно не прив'язане ніякого синтаксису.

    cp /usr/share/mc/syntax/sh.syntax /usr/share/mc/syntax/unknown.syntax

Надати права `sudo` для користувача

    usermod -aG sudo username

## Налаштування брандмауера
Встановити UFW, це зручний інтерфейс `iptables`.
        
    apt install ufw
        
**Увага!** Дозволити доступ по `ssh`, щоб не втратити можливість віддаленого управління сервером: 

    ufw allow ssh
        
Якщо для SSH змінено стандартний порт, наприклад, на **2233**, тоді команда буде мати наступний вигляд:
        
    ufw allow 2233/tcp

Відкрити порт `80(http)` та `443(https)`
    
    ufw allow http
    ufw allow https
    
Включити брандмауер:

    ufw enable
        
Переконатися, що з'єднання `SSH` не блокують брандмауером:        

    ufw status
        
**Перш ніж відключатись від користувача `root` перевірити чи можна підключитися по `SSH` звичайним користувачем!**  
Далі всі команди запускати від звичайного користувача використовуючи попереду команд `sudo`

## Встановлення Apache

    sudo apt install -y apache2 apache2-utils libapache2-mod-php

Після встановлення сервіс автоматично запуститься.  
Додати Apach в автозавантаження системи.
        
    sudo systemctl enable apache2

Дізнатись ip адресу однією з цих команд

    ip -4 a
    hostname -I
    curl http://ifconfig.co

Перевірити чи працює. В адресному рядку браузера ввести ip адресу сервера:

    links http://server_ip        

Має відкритись сторінка встановленого Apach


## Встановлення MariaDB

    sudo apt install -y mariadb-server mariadb-client
    
Перевірити версію MariaDB    

    mariadb --version
    
Запустити сценарій безпеки, який видалить ненадійні параметри і захистить БД від несанкціонованого доступу.    

    sudo mysql_secure_installation
    
Ввесити пароль `root` потім `N` а на інші питання `Y`

## Встановлення PHP

    sudo apt install -y php php-mysql php-curl php-gd php-mbstring php-xml php-xmlrpc php-soap php-intl php-zip php-cli php-json php-pear php-bcmath
    php -v
    
Щоб веб-сервер обслуговував PHP-файли першими, відредагувати файл `dir.conf`

    sudo nano /etc/apache2/mods-enabled/dir.conf

Перемістити `index.php` в початок рядка `DirectoryIndex`  
Зберегти `Ctrl+O` та закрити `Ctrl+X`  
Перезапустіти Apache і перевірити його стан

    sudo systemctl restart apache2.service
    sudo systemctl status apache2.service
    
## Створення бази даних MariaDB

    sudo mariadb

Створити базу даних з назвою `mydatabase`

    CREATE DATABASE mydatabase DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;

Створити користувача `secret_user` для бази даних `mydatabase`. Замість `12345qwerty` придумати складний пароль.

    GRANT ALL ON mydatabase.* TO 'secret_user'@'localhost' IDENTIFIED BY '12345qwerty';

Оновити привілеї

    FLUSH PRIVILEGES;

Перевірити результат

    SHOW DATABASES;
    EXIT;
    
## Налаштування віртуального хоста Apache
Тестовий домен `test.com`  
Видалити каталог `/var/www/html`

    sudo rm -ri /var/www/html

Закоментувати ВСЕ в файлі `/etc/apache2/sites-available/000-default.conf`

    sudo nano /etc/apache2/sites-available/000-default.conf

Створити каталог для тестового доменну

    sudo mkdir /var/www/test.com
    
Створити новий файл конфігурації що описує домен

    sudo nano /etc/apache2/sites-available/test.com.conf

Вставити такий текст
```
<VirtualHost *:80>
    # Основний (первинний) домен
    ServerName test.com
    # Список вторинних доменів (синоніми)
    ServerAlias www.test.com
    # Електронна пошта сервер-адміністратора
    ServerAdmin admin@example.com
    # Сховати версію веб-серверу від зловмисників
    ServerSignature Off
    # Параметри журналювання
    ErrorLog ${APACHE_LOG_DIR}/test.com-error.log
    CustomLog ${APACHE_LOG_DIR}/test.com-access.log combined
    LogLevel  warn
    # Каталог с файлами интернет-магазина
    DocumentRoot /var/www/test.com
    # Визначаємо правила поведінки контенту
    <Directory />
        Options +Indexes +FollowSymLinks +MultiViews
        AllowOverride All
    </Directory>
    <Directory /var/www/test.com>
        Options +Indexes +FollowSymLinks +MultiViews
        AllowOverride All
        Order allow,deny
        Allow from all
    </Directory>
</VirtualHost>
```
Перевірити синтаксис

    sudo apache2ctl configtest

Увімкнути підтримку `rewrite`

    sudo a2enmod rewrite
    
Підключити `php` як модулю `Apach`

    sudo a2enmod php7.3
    
Активувати віртуалхост `test.com`

    sudo a2ensite test.com

Перезапустити `Apache`

    systemctl restart apache2.service
    
### Тест PHP
В кореневій директорії для домену `test.com`, створити файл `info.php` 

     sudo nano /var/www/test.com/info.php

з таким змістом
```php
<?php
    phpinfo();
```

Відкрити в браузері сторінку `test.com/info.php`  

    links test.com/info.php
    
Має відкритись сторінка де буде вся інформація про встановлений PHP
___
**Після тестування обов'язково видалити файл info.php, він не повинен бути у відкритому доступі!**
___

## Встановлення та налаштування Wordpress



    

        
