# Создание веб-сервера. Команды для лекции.

Полезные ссылки:
<https://www.digitalocean.com/community/tutorials/how-to-install-linux-nginx-mysql-php-lemp-stack-on-ubuntu-20-04-ru>
<https://www.digitalocean.com/community/tutorials/how-to-install-wordpress-with-lemp-on-ubuntu-18-04-ru>
<https://www.nginx.com/blog/installing-wordpress-with-nginx-unit/>

Добавляем нового пользователя
```
adduser  <username>
```
Добавляем пользователя в группу sudo
```
usermod -aG sudo <username>
```
Настраиваем SSH доступ
Добавляем всем пользователям SSH-ключи. Проверяем доступ.
```
ssh-copy-id <username>@<host IP>
```
Меняем стандартный порт для доступа по SSH
Меняем порт в файле
```
/etc/ssh/sshd_config*
```
Раскомментируем строку ‘Port 22’. Вместо 22 укажите другой номер — например, ‘Port 56713’. Сохраните изменения и закройте файл.

Ищем в том же файле строку **PasswordAuthentication yes** меняем yes на no
Перезапускаем сервис SSH
```
systemctl restart ssh
```

## Устанавливаем стек LEMP

### Ставим nginx

```
sudo apt install nginx
```

Используем стандартный файервол линукс. Разрешаем NGNIX трафик на 80 и 443 порту.
```
sudo ufw allow 'Nginx Full'
```
Проверяем работу сервера, заходим на сервер по IP или домену

### Ставим базу данных
```
sudo apt install mysql-server
```
Изменяем метод аутентификации рута. С помошью первой команды заходим в интерфейс сервера Mysql
Вторая команда меняет метод аутентификации, присваивая пользователю root пароль qwerty. Пароль разумеется нежно использовать надежный, qwerty тут только для примера.
Третья команда обновляет привилегии пользователей

```
sudo mysql -u root

ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'qwerty';

FLUSH PRIVILEGES;
```

Настраиваем безопасность MySQL
```
sudo mysql_secure_installation
```
Сервис спросит, нужно ли использовать плагин VALIDATE PASSWORD PLUGIN. Он проверяет пароли на надежность. Если его использовать, то он запросит уровни сложности пароля от 1 до 3. И когда будете вводить пароль, будет проверять их на валидность. Можно не использовать, если вы итак задаете достаточно надежный пароль.

Далее задаем пароль рута для базы данных

Заходим в консоль управления БД
```
sudo mysql
```

### Ставим PHP
```
sudo apt install php-fpm php-mysql php-curl php-gd php-intl php-mbstring php-soap php-xml php-xmlrpc php-zip
sudo systemctl restart php7.4-fpm
```
LEMP установлен.

## Настройка LEMP

Логинимся под рутом в сервере Mysql
```
mysql -u root -p
```
## Создаем БД
```
CREATE DATABASE wordpress DEFAULT CHARACTER SET utf8 COLLATE utf8_unicode_ci;
```

Создаем пользователя БД. wpuser - имя юзера, password - пароль.
```
CREATE USER 'wpuser'@'localhost' IDENTIFIED BY 'password';
```
Даем ему привилегии. wordpress - имя базы данных, wpuser - имя юзера
```
GRANT ALL PRIVILEGES ON wordpress . * TO 'wpuser'@'localhost';
```
```
FLUSH PRIVILEGES;
```
Для выходы из консоли Mysql вводим
```
exit
```

база готова.

## Настрйка NGINX.

Мы будем использовать директорию по умолчанию, иначе это растянется на долго. Но в nginx можно настроить нескольок дирректорий и разместить на одном сервере несколько приложений.

Настройки дефолтной директории находятся по адресу. Открываем с sudo
```
sudo nano /etc/nginx/sites-available/default`
```

Редактируем блок server. Добавляем пути до фавикона и файла роботс.тхт

```
server {
    . . .

    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt { log_not_found off; access_log off; allow all; }
    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        expires max;
        log_not_found off;
    }
    . . .
}
```

Внутри существующего блока `location /` нам нужно изменить список `try_files`, чтобы вместо возвращения ошибки 404 по умолчанию, управление передавалось файлу `index.php` с аргументами запроса.

Выглядеть должно примерно так:

```
server {
    . . .
    location / {
        #try_files $uri $uri/ =404;
        try_files $uri $uri/ /index.php$is_args$args;
    }
    . . .
}
```

Добавляем еще один блок Locations чтобы работал PHP

```
  location ~ \.php$ {
         include snippets/fastcgi-php.conf;
         fastcgi_pass unix:/var/run/php/php7.4-fpm.sock;
         fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
         include fastcgi_params;
    }
```

Добавляем в конфиг чтение файла index.php

Проверяем файл конфигурации на ошибки
```
sudo nginx -t
```
Если все ок, прерзагружаем вебсервер
```
sudo systemctl reload nginx
```
Вебсервер настроен

Качаем вордпрес, распаковываем архив
```
wget <https://ru.wordpress.org/latest-ru_RU.tar.gz>
```
```
tar xzvf latest-ru_RU.tar.gz
```
создаем файл настроек
```
cp \~/wordpress/wp-config-sample.php \~/wordpress/wp-config.php
```
Копируем файлы вордпресс в основную папку
```
sudo cp -a \~/wordpress/.  /var/www/html
```
Даем права на файлы пользователю nginx
```
sudo chown -R www-data:www-data /var/www/html
```
генерируем ключи для вордпресс
```
sudo apt install curl
curl -s <https://api.wordpress.org/secret-key/1.1/salt/>
```
и вставляем в файл /var/www/html/wp-config.php

Добавим конфиги базы данных

И завершаем установку через браузер

Установка Cerbot
```
<https://certbot.eff.org/instructions?ws=nginx&os=ubuntufocal>
```
Установка PHPMyAdmin
```
<https://losst.pro/ustanovka-phpmyadmin-s-nginx-v-ubuntu-20-04?ysclid=ldzr8cw0d2675348461>
```
