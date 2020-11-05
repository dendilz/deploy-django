## Деплой Django приложения

### Создаёте пользователя с паролем
```
# useradd -s /bin/bash -G sudo -m django
# passwd django
# su django
```
### Установка git

```
$ sudo apt install git
```

### Установка pyenv и virtualenv
Сначала ставите необходимые зависимости
```
$ sudo apt-get install -y make build-essential libssl-dev
 zlib1g-dev libbz2-dev libreadline-dev libsqlite3-dev
 wget curl llvm libncurses5-dev libncursesw5-dev
 xz-utils tk-dev libffi-dev
```
Устанавливате pyenv
```
$ git clone https://github.com/pyenv/pyenv.git ~/.pyenv
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
$ echo -e 'if command -v pyenv 1>/dev/null 2>&1; then\n  eval "$(pyenv init -)"\nfi' >> ~/.bashrc
```
Устанавливаете pyenv-virtualenv
```
$ git clone https://github.com/pyenv/pyenv-virtualenv.git $(pyenv root)/plugins/pyenv-virtualenv; 
$ echo 'eval "$(pyenv virtualenv-init -)"' >> ~/.bashrc
```
### Установка Python для виртуального окружения
```
$ pyenv install 3.7.1
```
Создаёте окружение
```
$ pyenv virtualenv 3.7.1 env
```
### Установка проекта
```
$ git clone https://github.com/gothinkster/django-realworld-example-app
$ cd django-realworld-example-app/
$ pyenv local env
```
Добавляете в файл ```requirements.txt``` пакет ```psycopg2==2.8.6``` и устанавливаете зависимости
```
$ pip install -r requirements.txt
```

### Установка PostgreSQL
```
$ sudo apt install postgresql-server-dev-10
$ su postgres
$ psql
```
Создаём базу данных, пользователя:

```
> CREATE DATABASE django;
> CREATE USER django WITH PASSWORD 'password';
> GRANT ALL PRIVILEGES ON DATABASE django TO django;
```
Перезаходите под пользователем django
```
su django
```
### Установка gunicorn и создание сервиса

```
$ pip install gunicorn
$ sudo nano /etc/systemd/system/gunicorn.service
```
Пишем туда
```
Unit]
Description=gunicorn daemon
After=network.target

[Service]
User=django
Group=www-data
WorkingDirectory=/home/django/django-realworld-example-app
ExecStart=/home/django/.pyenv/shims/gunicorn --access-logfile - --workers 3 --bind unix:/home/django/django-realworld-example-app/conduit/conduit.sock conduit.wsgi:application

[Install]
WantedBy=multi-user.target

```

Включаете службу и добавляете в автозапуск
```
$ sudo systemctl start gunicorn
$ sudo systemctl enable gunicorn
```

### Настройка Nginx

```
$ sudo nano /etc/nginx/sites-enabled/le
```
Пишем туда
```
add_header X-Frame-Options SAMEORIGIN;
add_header X-Content-Type-Options nosniff;
add_header X-XSS-Protection "1; mode=block";
add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline' 'unsafe-eval' https://www.google-analytics.com; img-src 'self' data: https://www.google-analy$


# HTTPS server
#
server {
    listen 443 ssl default deferred;
    server_name dendilz-django.devops.rebrain.srwx.net;

    ssl on;
    ssl_certificate         /etc/letsencrypt/live/dendilz-django.devops.rebrain.srwx.net/fullchain.pem;
    ssl_certificate_key     /etc/letsencrypt/live/dendilz-django.devops.rebrain.srwx.net/privkey.pem;
    ssl_trusted_certificate /etc/letsencrypt/live/dendilz-django.devops.rebrain.srwx.net/fullchain.pem;

    ssl_session_cache shared:SSL:50m;
    ssl_session_timeout 5m;
    ssl_stapling on;
    ssl_stapling_verify on;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-$

    ssl_dhparam /etc/nginx/dhparams.pem;
    ssl_prefer_server_ciphers on;

    root /home/django/django-realworld-example-app/conduit;
    index index.html index.htm;

    location / {
      include proxy_params;
      proxy_pass http://unix:/home/django/django-realworld-example-app/conduit/conduit.sock;
    }

    location /static/ {
      root /home/django/django-realworld-example-app/;
    }
}
```
Перезапускаете nginx
```
$ sudo service nginx reload
```

### Настраиваем проект
```
$ nano django-realworld-example-app/conduit/settings.py 
```
Меняем в нём настройки подключения к БД

```
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql_psycopg2',
        'NAME': 'django',
        'USER' : 'django',
        'PASSWORD' : 'password',
        'HOST' : '127.0.0.1',
        'PORT' : '5432',
    }
}
```
И после ```STATIC_URL = '/static/'``` добавляете
```
STATIC_ROOT = os.path.join(BASE_DIR, 'static/')
```
Сохраняем, делаем миграции и создаём статику:
```
$ python manage.py makemigrations
$ python manage.py migrate
$ python manage.py collectstatic
```
### Проверка
[Django-Project](https://dendilz-django.devops.rebrain.srwx.net/api/)
