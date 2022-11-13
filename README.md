<h3>### DYNAMIC WEB ###</h3>

<p>развертывание веб приложения</p>

<h4>Описание домашнего задания</h4>

<ul>Варианты стенда:
<li>nginx + php-fpm (laravel/wordpress) + python (flask/django) + js(react/angular);</li>
<li>nginx + java (tomcat/jetty/netty) + go + ruby;</li>
<li>можно свои комбинации.</li>
</ul>

<ul>Реализации на выбор:
<li>на хостовой системе через конфиги в /etc;</li>
<li>деплой через docker-compose.</li>
</ul>

<p>Для усложнения можно попросить проекты у коллег с курсов по разработке</p>

<ul>К сдаче принимается:
<li>vagrant стэнд с проброшенными на локалхост портами</li>
<li>каждый порт на свой сайт</li>
<li>через nginx</li>
</ul>

<p>Формат сдачи ДЗ - vagrant + ansible</p>

<h4>Создание стенда "Dynamic web"</h4>

<p>Содержимое Vagrantfile:</p>

<pre>[user@localhost dynamic_web]$ <b>vi ./Vagrantfile</b></pre>

<pre># -*- mode: ruby -*-
# vi: set ft=ruby :

MACHINES = {
  :dynweb => {
    :box_name => "centos/7",
    :vm_name => "dynweb",
    :ip => '192.168.50.10', # for ansible
    :mem => '2048',
    :cpus => '2'
  }
}
Vagrant.configure("2") do |config|
  MACHINES.each do |boxname, boxconfig|
    config.vm.define boxname do |box|
      box.vm.box = boxconfig[:box_name]
      box.vm.host_name = boxname.to_s
      box.vm.network "private_network", ip: boxconfig[:ip]
      box.vm.network "forwarded_port", guest: 8081, host: 8081
      box.vm.network "forwarded_port", guest: 8082, host: 8082
      box.vm.network "forwarded_port", guest: 8083, host: 8083
      box.vm.provider :virtualbox do |vb|
        vb.customize ["modifyvm", :id, "--memory", boxconfig[:mem]]
        vb.customize ["modifyvm", :id, "--cpus", boxconfig[:cpus]]
      end
#      if boxconfig[:vm_name] == "dynweb"
#        box.vm.provision "ansible" do |ansible|
#          ansible.playbook = "ansible/playbook.yml"
#          ansible.inventory_path = "ansible/hosts"
#          ansible.become = true
#          ansible.host_key_checking = "false"
#          ansible.limit = "all"
#          ansible.verbose = "vvv"
#        end
#      end
    end
  end
end</pre>

<p>Запустим виртуальную машину:</p>

<pre>[user@localhost dynamic_web]$ <b>vagrant up</b></pre>

<pre>[user@localhost dynamic_web]$ <b>vagrant status</b>
Current machine states:

dynweb                    running (virtualbox)

This environment represents multiple VMs. The VMs are all listed
above with their current state. For more information about a specific
VM, run `vagrant status NAME`.
[user@localhost dynamic_web]$</pre>

<pre>[user@localhost dynamic_web]$ <b>vagrant ssh dynweb</b>
[vagrant@dynweb ~]$ <b>sudo -i</b>
[root@dynweb ~]#</pre>

<p>мы рассмотрим вариант стенда nginx + php-fpm (wordpress) + python (django) + js(node.js) с деплоем через docker-compose.</p>

<p>Настройка окружения в docker-compose</p>

<p>Установим пакет yum-utils:</p>

<pre>[root@dynweb ~]# <b>yum install yum-utils -y</b></pre>

<p>Добавим официальный репозиторий Docker в систему:</p>

<pre>[root@dynweb ~]# <b>yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo</b></pre>

<p>Установим Docker:</p>

<pre>[root@dynweb ~]# <b>yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin -y</b></pre>

<p>Смотрим, в какие группы входит пользователь <i>vagrant</i>:</p>

<pre>[root@dynweb ~]# <b>id vagrant</b>
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant)
[root@dynweb ~]#</pre>

<p>Добавим пользователя <i>vagrant</i> в группу <i>docker</i>:

<pre>[root@dynweb ~]# usermod -aG docker vagrant
[root@dynweb ~]#</pre>

<p>Убеждаемся, что пользователь <i>vagrant</i> уже входит в группу <i>docker</i>:</p>

<pre>[root@dynweb ~]# <b>id vagrant</b>
uid=1000(vagrant) gid=1000(vagrant) groups=1000(vagrant),<b>993(docker)</b>
[root@dynweb ~]#</pre>

<p>Создадим директорий <i>docker</i>:</p>

<pre>[root@dynweb ~]# <b>mkdir ./docker</b>
[root@dynweb ~]#</pre>

<p>Переходим в этот директорий:</p>

<pre>[root@dynweb ~]# <b>cd ./docker/</b>
[root@dynweb docker]#</pre>

<p>Теперь будем созавать файл <i>docker-compose.yml</i>.</p>

<p>Для развёртки wordpress необходима база данных, выберем <i>mysql</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
database:
  image: mysql:8.0 # используем готовый образ mysql от разработчиков
  container_name: database
  restart: unless-stopped
  environment:
    MYSQL_DATABASE: ${DB_NAME} # Имя и пароль базы данных будут задаваться в отдельном .env файле
    MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
  volumes:
  - ./dbdata:/var/lib/mysql # Чтобы данные базы не пропали при остановке/удалении контейнера, будем сохранять их на хост-машине
  command: '--default-authentication-plugin=mysql_native_password'</pre>

<p>Создаём файл переменных <i>.env</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./.env</b>
# Переменные которые будут использоваться для создания и подключения БД
DB_NAME=wordpress
DB_ROOT_PASSWORD=dbpassword
# Переменные необходимые python приложению
MYSITE_SECRET_KEY=put_your_django_app_secret_key_here
DEBUG=True</pre>

<p>Для того чтобы объединить наши приложения, создадим сеть и будем добавлять каждый контейнер в неё:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
  database:
    image: mysql:8.0 <i># используем готовый образ mysql от разработчиков</i>
    container_name: database
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${DB_NAME} </i># Имя и пароль базы данных будут задаваться в отдельном .env файле</i>
      MYSQL_ROOT_PASSWORD: ${DB_ROOT_PASSWORD}
    volumes:
    - ./dbdata:/var/lib/mysql <i># Чтобы данные базы не пропали при остановке/удалении контейнера, будем сохранять их на хост-машине</i>
    command: '--default-authentication-plugin=mysql_native_password'
<b>    networks:
    - app-network

networks:
  app-network:
    driver: bridge</b></pre>

<p>Контейнер wordpress:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
...
  wordpress:
    image: wordpress:5.1.1-fpm-alpine <i># официальный образ от разработчиков</i>
    container_name: wordpress
    restart: unless-stopped
    <i># на странице образа в docker hub написано, какие можно задать переменные контейнеру https://hub.docker.com/_/wordpress</i>
    environment:
      WORDPRESS_DB_HOST: database
      WORDPRESS_DB_NAME: "${DB_NAME}" <i># Также импортируем переменные из .env</i>
      WORDPRESS_DB_USER: root
      WORDPRESS_DB_PASSWORD: "${DB_ROOT_PASSWORD}"
    volumes:
    - ./wordpress:/var/www/html <i># сохраняем приложение на хост машине</i>
    networks:
    - app-network
    depends_on:
    - database <i># контейнер wordpress дождется запуска БД</i></pre>

<p>Контейнер nginx:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
...
  nginx:
    image: nginx:1.15.12-alpine
    container_name: nginx
    restart: unless-stopped
    <i># Т.к. все запросы к приложениям будут проходить через nginx, пробросим под каждое приложение по порту.</i>
    ports:
    - 8081:8081
    - 8082:8082
    - 8083:8083
    volumes:
    <i># будет использоваться php-fpm, необходимо смонтировать статические файлы wordpress:</i>
    - ./wordpress:/var/www/html
    - ./nginx:/etc/nginx/conf.d <i># монтируем конфиг</i>
    networks:
    - app-network
    depends_on: <i># nginx будет запускаться после всех приложений</i>
    - wordpress
    - app
    - node</pre>

<p>Рассмотрим сервер nginx для wordpress.<br />
Создаём директорий <i>nginx</i> для размещения nginx конфиг файла:</p>

<pre>[root@dynweb docker]# <b>mkdir ./nginx</b>
[root@dynweb docker]#</pre>

<p>Создадим nginx конфиг файл:</p>

<pre>[root@dynweb docker]# <b>vi ./nginx/nginx.conf</b>
<i># Сервер nginx для wordpress:</i>
<i># Данный сервер отвечает за проксирование на wordpress через fastcgi</i>
server {
<i># Wordpress будет отображаться на 8081 порту хоста</i>
        listen 8081;
        listen [::]:8081;
        server_name localhost;
        index index.php index.html index.htm;
<i># Задаем корень корень проекта, куда мы смонтировали статику wordpress</i>
        root /var/www/html;
        location ~ /.well-known/acme-challenge {
                allow all;
                root /var/www/html;
        }
        location / {
                try_files $uri $uri/ /index.php$is_args$args;
        }
<i># Само fastcgi проксирование в контейнер с wordpress по 9000 порту</i>
        location ~ \.php$ {
                try_files $uri =404;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_pass wordpress:9000;
                fastcgi_index index.php;
                include fastcgi_params;
                fastcgi_param SCRIPT_FILENAME
                $document_root$fastcgi_script_name;
                fastcgi_param PATH_INFO $fastcgi_path_info;
        }
        location = /favicon.ico {
                log_not_found off; access_log off;
        }
        location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
                expires max;
                log_not_found off;
        }
}</pre>

<p>Сервер nginx для django:</p>

<pre><i># Сервер nginx для django:</i>
upstream django {
    server app:8000;
}
server {
<i># Django будет отображаться на 8082 порту хоста</i>
        listen 8082;
        listen [::]:8082;
        server_name localhost;
        location / {
                try_files $uri @proxy_to_app;
        }
<i># тут используем обычное проксирование в контейнер django</i>
        location @proxy_to_app {
                proxy_pass http://django;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Host $server_name;
        }
}</pre>

<p>Сервер nginx для node.js:</p>

<pre><i># Сервер nginx для node.js:</i>
<i># Node.js будет отображаться на 8083 порту хоста</i>
server {
        listen 8083;
        listen [::]:8083;
        server_name localhost;
        location / {
                proxy_pass http://node:3000;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection "upgrade";
                proxy_redirect off;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header X-Forwarded-Host $server_name;
        }
}</pre>

<p>Описание контейнера django:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
...
  app:
    build: ./django <i># для нашего приложения нужны зависимости, поэтому собираем свой образ</i>
    container_name: app
    restart: always
    env_file:
    - .env <i># импортируем в контейнер переменные из .env</i>
    command:
      "gunicorn --workers=2 --bind=0.0.0.0:8000 mysite.wsgi:application" <i># команда для запуска django проекта, приложение будет работать на 8000 порту контейнера</i>
    networks:
    - app-network</pre>

<p>Исходный код Django приложения.</p>

<p>Создаём директорий <i>django</i> для размещения python файлов:</p>

<pre>[root@dynweb docker]# <b>mkdir ./django</b>
[root@dynweb docker]#</pre>

<p>Создадим файл <i>requirements.txt</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/requirements.txt</b></p>
Django==3.1
gunicorn==20.0.4
pytz==2020.1</pre>

<p>Создадим файл <i>manage.py</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/manage.py</b></p>
#!/usr/bin/env python
"""Django's command-line utility for administrative tasks."""
import os
import sys

def main():
    """Run administrative tasks."""
    os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')
    try:
        from django.core.management import execute_from_command_line
    except ImportError as exc:
        raise ImportError(
            "Couldn't import Django. Are you sure it's installed and "
            "available on your PYTHONPATH environment variable? Did you "
            "forget to activate a virtual environment?"
        ) from exc
    execute_from_command_line(sys.argv)
if __name__ == '__main__':
    main()</pre>

<p>В директории <i>django</i> создадим директорий <i>mysite</i>:</p>

<pre>[root@dynweb docker]# <b>mkdir ./django/mysite</b>
[root@dynweb docker]#</pre>

<p>В этом директории создадим файл <i>wsgi.py</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/mysite/wsgi.py</b></p>
import os
from django.core.wsgi import get_wsgi_application
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'mysite.settings')
application = get_wsgi_application()</pre>

<p>Создадим файл <i>urls.py</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/mysite/urls.py</b></p>
from django.contrib import admin
from django.urls import path

urlpatterns = [
    path('admin/', admin.site.urls),
]</pre>

<p>Создадим файл <i>settings.py</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/mysite/settings.py</b></p>
import os
import ast
from pathlib import Path

BASE_DIR = Path(__file__).resolve(strict=True).parent.parent
SECRET_KEY = os.getenv('MYSITE_SECRET_KEY', '')
DEBUG = ast.literal_eval(os.getenv('DEBUG', 'True'))
ALLOWED_HOSTS = []
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
ROOT_URLCONF = 'mysite.urls'
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
WSGI_APPLICATION = 'mysite.wsgi.application'
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.sqlite3',
        'NAME': BASE_DIR / 'db.sqlite3',
    }
}
AUTH_PASSWORD_VALIDATORS = [
    {
        'NAME': 'django.contrib.auth.password_validation.UserAttributeSimilarityValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.MinimumLengthValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.CommonPasswordValidator',
    },
    {
        'NAME': 'django.contrib.auth.password_validation.NumericPasswordValidator',
    },
]
LANGUAGE_CODE = 'en-us'
TIME_ZONE = 'UTC'
USE_I18N = True
USE_L10N = True
USE_TZ = True
STATIC_URL = '/static/'</pre>

<p>Описание контейнера node.js:</p>

<pre>[root@dynweb docker]# <b>vi ./docker-compose.yml</b>
...
  node:
    image: node:16.13.2-alpine3.15
    container_name: node
    working_dir: /opt/server <i># переназначим рабочую директорию для удобства</i>
    volumes:
    - ./node:/opt/server <i># пробрасываем приложение в директорию контейнера</i>
    command: node test.js <i># запуск приложения</i>
    networks:
    - app-network</pre>

<p>Сам <i>Dockerfile</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./django/Dockerfile</b>
FROM python:3.8.3
ENV APP_ROOT /src
ENV CONFIG_ROOT /config
RUN mkdir ${CONFIG_ROOT}
COPY requirements.txt ${CONFIG_ROOT}/requirements.txt
RUN pip install -r ${CONFIG_ROOT}/requirements.txt
RUN mkdir ${APP_ROOT}
WORKDIR ${APP_ROOT}
ADD . ${APP_ROOT}</pre>

<p>Исходный код node.js приложения:</p>

<p>Создаём директорий <i>node</i> для размещения node.js файлов:</p>

<pre>[root@dynweb docker]# <b>mkdir ./node</b>
[root@dynweb docker]#</pre>

<p>Создадим файл <i>test.js</i>:</p>

<pre>[root@dynweb docker]# <b>vi ./node/test.js</b>
const http = require('http');
const hostname = '0.0.0.0';
const port = 3000;

const server = http.createServer((req, res) => {
    res.statusCode = 200;
    res.setHeader('Content-Type', 'text/plain');
    res.end('Hello from node js server');
});
server.listen(port, hostname, () => {
    console.log(`Server running at http://${hostname}:${port}/`);
});</pre>

<p>Полученная структура каталогов и файлов:</p>

<pre>[root@dynweb docker]# <b>tree</b>
.
├── django
│   ├── Dockerfile
│   ├── manage.py
│   ├── mysite
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── requirements.txt
├── docker-compose.yml
├── nginx
│   └── nginx.conf
└── node
    └── test.js

4 directories, 9 files
[root@dynweb docker]#</pre>

<p>Запускаем docker сервис:</p>

<pre>[root@dynweb docker]# <b>systemctl start docker</b>
[root@dynweb docker]# <b>systemctl status docker</b>
● docker.service - Docker Application Container Engine
   Loaded: loaded (/usr/lib/systemd/system/docker.service; disabled; vendor preset: disabled)
   Active: active (running) since Sat 2022-11-12 20:36:42 UTC; 4s ago
     Docs: https://docs.docker.com
 Main PID: 22678 (dockerd)
    Tasks: 7
   Memory: 23.8M
   CGroup: /system.slice/docker.service
           └─22678 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/con...

Nov 12 20:36:41 dynweb dockerd[22678]: time="2022-11-12T20:36:41.837290410Z...pc
Nov 12 20:36:41 dynweb dockerd[22678]: time="2022-11-12T20:36:41.837307994Z...pc
Nov 12 20:36:41 dynweb dockerd[22678]: time="2022-11-12T20:36:41.837317584Z...pc
Nov 12 20:36:41 dynweb dockerd[22678]: time="2022-11-12T20:36:41.854869214Z...."
Nov 12 20:36:42 dynweb dockerd[22678]: time="2022-11-12T20:36:42.007268999Z...s"
Nov 12 20:36:42 dynweb dockerd[22678]: time="2022-11-12T20:36:42.068446952Z...."
Nov 12 20:36:42 dynweb dockerd[22678]: time="2022-11-12T20:36:42.081683229Z...21
Nov 12 20:36:42 dynweb dockerd[22678]: time="2022-11-12T20:36:42.082022750Z...n"
Nov 12 20:36:42 dynweb dockerd[22678]: time="2022-11-12T20:36:42.102607680Z...k"
Nov 12 20:36:42 dynweb systemd[1]: Started Docker Application Container Engine.
Hint: Some lines were ellipsized, use -l to show in full.
[root@dynweb docker]#</pre>

<p>Запустим docker-compose.yml:</p>

<pre>[root@dynweb docker]# <b>docker compose -f ./docker-compose.yml up -d</b></pre>

<p>Откроем браузер и в адресной строке вводим <b><i>127.0.0.1:8081</i></b>:</p>

<img src="./screens/Screenshot from 2022-11-13 00-33-02.png" alt="wordpress" />

<p>Вводим в адресной строке <b><i>127.0.0.1:8082</i></b>:</p>

<img src="./screens/Screenshot from 2022-11-13 00-33-13.png" alt="django" />

<p>И наконец в адресной строке вводим <b><i>127.0.0.1:8083</i></b>:</p>

<img src="./screens/Screenshot from 2022-11-13 00-33-24.png" alt="node.js" />

<h4>Запуск стенда "Dynamic Web"</h4>

<p>Запустить стенд с помощью следующей команды:</p>

<pre>$ git clone https://github.com/SergSha/dynamic_web.git && cd ./dynamic_web/ && vagrant up</pre>

<p>После завершения открываем браузер и в адресной строке вводим:<br />

<pre>localhost:8081</pre>

<p>Откроется стартовая страница wordpress;</p>

<pre>localhost:8082</pre>

<p>Откроется стартовая страница django;</p>

<pre>localhost:8083</pre>

<p>Откроется тестовая страница node.js.</p>

