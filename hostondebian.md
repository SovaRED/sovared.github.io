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
    
Включити підсвічування синтаксису всіх файлів, які не позначені явно в файлі `/usr/share/mc/syntax/Syntax` синтаксисом для `sh` і `bash` скриптів. Цей універсальний синтаксис нормально підходить для конфігураційних файлів, з якими найчастіше доводиться працювати на сервері. Перезаписати файл `unknown.syntax`. Саме цей шаблон буде застосовуватися до `.conf` і `.cf` файлів, так як до них явно не прив'язане ніякого синтаксису.

    cp /usr/share/mc/syntax/sh.syntax /usr/share/mc/syntax/unknown.syntax

Надати права `sudo` для користувача

    usermod -aG sudo username

## Налаштування брандмауера
Встановити UFW - це зручний інтерфейс `iptables`.
        
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
        
Переконатися, що з'єднання `SSH` не блокується брандмауером:        

    ufw status
        
**Перш ніж відключатись від користувача `root` перевірити чи можна підключитися по `SSH` звичайним користувачем!**  
Далі всі команди запускати від звичайного користувача використовуючи попереду команд `sudo`

## Встановлення Apache

    sudo apt install -y apache2 apache2-utils libapache2-mod-php

Після встановлення сервіс автоматично запуститься.  
Додати Apache в автозавантаження системи.
        
    sudo systemctl enable apache2

Дізнатись ip адресу однією з цих команд

    ip -4 a
    hostname -I
    curl http://ifconfig.co

Перевірити чи працює Apache. В адресному рядку браузера ввести ip адресу сервера:

    links http://server_ip        

Має відкритись сторінка встановленого Apache


## Встановлення MariaDB

    sudo apt install -y mariadb-server mariadb-client
    
Перевірити версію MariaDB    

    mariadb --version
    
Запустити сценарій безпеки, який видалить ненадійні параметри і захистить БД від несанкціонованого доступу.    

    sudo mysql_secure_installation
    
Ввесити пароль для `root` потім `N` а на інші питання `Y`

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
    
Підключити `php` як модулю `Apache`

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

**Після тестування обов'язково видалити файл info.php, він не повинен бути у відкритому доступі!**


## Встановлення та налаштування Wordpress

Перейти в каталог `/tmp`, скачати в нього свіжу версію `Wordpress` й розархівувати

        cd /tmp
        curl -O https://wordpress.org/latest.tar.gz
        tar xzvf latest.tar.gz
        
Скопіювати вміст в кореневу директорію сайту        

    sudo cp -a /tmp/wordpress/. /var/www/test.com/
    
Подальші налаштування будуть відносно кореневого каталогу сайта `/var/www/test.com/`

    cd /var/www/test.com

Створити порожній файл `.htaccess`

    sudo touch .htaccess
    
Створити каталог `upgrade`

    sudo mkdir /var/www/test.com/wp-content/upgrade

Скопіювати файл `wp-config-sample.php` в файл `wp-config.php`

    sudo cp /var/www/test.com/wp-config-sample.php /var/www/test.com/wp-config.php
    
Встановити правильні права доступу на каталоги. Користувач і групи повинні бути `www-data`

    sudo chown -R www-data:www-data /var/www/test.com/
    
За правилами безпеки, права на каталоги повинні бути `755`, а на файли `644`

    sudo find /var/www/test.com/ -type d -exec chmod 755 {} \;
    sudo find /var/www/test.com/ -type f -exec chmod 644 {} \;
    
Перевірити

    ls -la
    
Згенерувати ключі безпеки для шифрування інформації яка зберігається в `Cookies`

    curl -s https://api.wordpress.org/secret-key/1.1/salt/
    
Зберегти автоматично згенерували ключі.  
Вони мають мати схожий вигляд. 

![gen-key](https://cdn.freehost.com.ua/wpdeb07.jpg "Вигляд згенерованих ключів")

Відкрити `wp-config.php`

    sudo nano /var/www/test.com/wp-config.php
    
Знайти наступні рядки

```php
define( 'AUTH_KEY', 'put your unique phrase here' );
define( 'SECURE_AUTH_KEY', 'put your unique phrase here' );
define( 'LOGGED_IN_KEY', 'put your unique phrase here' );
define( 'NONCE_KEY', 'put your unique phrase here' );
define( 'AUTH_SALT', 'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT', 'put your unique phrase here' );
define( 'NONCE_SALT', 'put your unique phrase here' );
```
Видалити ці рядки, і на їх місце вставити ті рядки що були згенеровані раніше  

Також в файлі `wp-config.php` налаштувати підключення до бази даних, використовуючи дані користувача, якого створили раніше.  

![databaseconnect](https://cdn.freehost.com.ua/wpdeb08.jpg "Вигляд налаштованого підключення до бази")
    
Зберегти `wp-config.php`.  

Тепер можна почати встановлення `Wordpress` через інтерфейс в браузері.  
Вибрати мову і натиснути «Продовжити».  
Далі заповнити всі поля та підтвердити натисканням `Встановити Wordpress`

## Налаштування SSL для домену на Wordpress

Отримувати безкоштовний `Let's Encrypt` сертифікат за допомогою утиліти `Certbot`  
Встановити `snapd` та оновити 

    sudo apt install snapd
    sudo snap install core
    sudo snap refresh core

Вилучити `certbot-auto` та всі пакунки `Certbot`

    sudo apt-get remove certbot

Встановити `Certbot`

    sudo snap install --classic certbot
    
Переконатися, що команда `certbot` може бути запущена

    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    
Отримати сертифікат, і `certbot` автоматично відредагує конфігурацію `Apache` ввімкнувши доступ `https`
    
    sudo certbot --apache
    
 Перевірити автоматичне поновлення сертифікатів можна так
 
    sudo certbot renew --dry-run
    
 `certbot` автоматично пропише себе в `cron` та буде автоматично оновлювати сертифікат якщо не буде змінено конфігурацію.
 
 `Certbot` автоматично все згенерував і розклав по каталогам.  
 Сертифікат (certificate) і ланцюжок (chain) зберіг в: `/etc/letsencrypt/live/test.com/fullchain.pem`  
 Приватний ключ згенеровано тут: `/etc/letsencrypt/live/test.com/privkey.pem`  
 Зручність полягає також у тому, що `Certbot` просканував всі наші каталоги і віртуалхости, автоматично створивши новий конфіг `test.com-le-ssl.conf`
 
    ls -la /etc/apache2/sites-available/
    
![config](https://cdn.freehost.com.ua/wpdeb14.jpg "")
 
Далі в панелі управління https://test.com/wp-admin, в лівому меню перейти в розділ `«Налаштування»`, в рядках `Адреса WordPress (URL)` і `Адреса сайту (URL)` змінити `http` на `https`. Натиснути `«Зберегти зміни»` знизу сторінки. 

![configwp](https://cdn.freehost.com.ua/wpdeb16.jpg "")

Відключаємо `HTTP`. Для цього файл `.htaccess` має виглядати так
```
<IfModule mod_rewrite.c>
    RewriteEngine On
    RewriteCond %{HTTPS} =off
    RewriteRule (.*) https://%{HTTP_HOST}%{REQUEST_URI} [QSA,L]
</IfModule>
```    
Якщо після цього виникне помилка циклічної переадресації, необхідно в файлі `wp-config.php`, після `<?Php` додати з нового рядка наступне 
```php
if ($_SERVER["HTTP_X_FORWARDED_PROTOCOL"] == 'https'){
    $_SERVER["HTTPS"]='on';
}
if ($_SERVER["HTTP_X_FORWARDED_PROTOCOL"] == 'http'){
    $_SERVER["HTTPS"]='on';
    header("Location: https://".$_SERVER["HTTP_HOST"].$_SERVER["REQUEST_URI"]);
    exit();
}
```

[УСТАНОВКА WORDPRESS НА DEBIAN 10. ИНСТРУКЦИЯ](https://freehost.com.ua/faq/articles/ustanovka-wordpress-na-debian-10-instruktsija/)

        
