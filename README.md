# SeaFile Writeup

I wanted to share some notes after strugling to setting up Seafile as a selfhosted Google Drive alternative. Selfhosting is nice if you value privacy of your files, and you want full control over your data. I landed on Seafile because it had a nice web GUI, had a free community version, and could be selfhosted in e.g. Docker.

In the end I didn't get to upload or download files because of strange errors such as:
1. "Network error" when dragging files into the web site for upload, and when viewing files I got the message `Sorry, but the requested page could not be found.`
2. Error in the seafile container when connecting to the seafile-mysql container `Skip running setup-seafile-mysql.py because there is existing seafile-data folder. Waiting for mysql server to be ready: %s (2003, "Can't connect to MySQL server on 'db' ([Errno 111] Connection refused)")`
3. Database error in the seafile-mysql container: `6 [Warning] Aborted connection 6 to db: 'seahub_db' user: 'seafile' host: '172.18.0.4' (Got an error reading communication packets)`

I don't know what's wrong, and I might look into it later out of curiosity. Here is my setup. I'm particulary happy i got the Nginx proxy in front to work. The purpose was to encrypt traffic.

### Nginx proxy and certificate setup
Commands used
````
rm /etc/nginx/sites-enabled/default && rm /etc/nginx/sites-available/default
touch /etc/nginx/sites-available/seafile.conf
ln -s /etc/nginx/sites-available/seafile.conf /etc/nginx/sites-enabled/seafile.conf

#certificates
sudo openssl dhparam -out /etc/nginx/dhparam.pem 2048
sudo openssl genrsa -out /etc/nginx/certz/seafile-privkey.pem 2048
sudo openssl req -new -x509 -key /etc/nginx/certz/seafile-privkey.pem -out /etc/nginx/certz/seafile-cacert.pem-days 365
#sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/certz/seafile-privkey.pem -out /etc/nginx/certz/seafile-cacert.pem
````

Nginx config for file `/etc/nginx/sites-enabled/seafile.conf` which i got from [this](https://manual.seafile.com/deploy/https_with_nginx/) site. [This](https://lins05.gitbooks.io/seafile-docs/content/deploy/https_with_nginx.html) one is also helpfu.
````
    server {
        listen       80;
        server_name  10.0.1.10;
        rewrite ^ https://$http_host$request_uri? permanent;    # Forced redirect from HTTP to HTTPS
        server_tokens off;
    }
    server {
        listen 443 ssl;
        ssl_certificate /etc/nginx/certz/seafile-cacert.pem;        # Path to your cacert.pem
        ssl_certificate_key /etc/nginx/certz/seafile-privkey.pem;   # Path to your privkey.pem
        server_name seafile.local;
        server_tokens off;

        # HSTS for protection against man-in-the-middle-attacks
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

        # DH parameters for Diffie-Hellman key exchange
        ssl_dhparam /etc/nginx/dhparam.pem;

        # Supported protocols and ciphers for general purpose server with good security and compatability with most clients
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_prefer_server_ciphers off;

        # Supported protocols and ciphers for server when clients > 5years (i.e., Windows Explorer) must be supported
        #ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
        #ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES256-SHA256:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA;
        #ssl_prefer_server_ciphers on;

        ssl_session_timeout 5m;
        ssl_session_cache shared:SSL:5m;

        location / {
            proxy_pass         http://127.0.0.1:8080;
            proxy_set_header   Host $http_host;
            proxy_set_header   X-Real-IP $remote_addr;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header   X-Forwarded-Host $server_name;
            proxy_set_header   X-Forwarded-Proto https;

            access_log      /var/log/nginx/seahub.access.log;
            error_log       /var/log/nginx/seahub.error.log;

            proxy_read_timeout  1200s;

            client_max_body_size 0;
        }

        location /seafhttp {
            rewrite ^/seafhttp(.*)$ $1 break;
            proxy_pass http://127.0.0.1:8080;
            client_max_body_size 0;
            proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_connect_timeout  36000s;
            proxy_read_timeout  36000s;
            proxy_send_timeout  36000s;
            send_timeout  36000s;
        }

#        location /media {
#            root /home/user/ExampleUser/seafile-server-latest/seahub;
#        }
    }
````


### Seafile in Docker setup


- Install docker
````
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update && sudo apt dist-upgrade
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl restart docker
````

docker compose setup file:
````
services:
  db:
    image: mariadb
    container_name: seafile-mysql
    environment:
      - MYSQL_ROOT_PASSWORD=CHANGE_THIS_TO_SUPER_SECRET_PW
      - MYSQL_LOG_CONSOLE=true
      - MARIADB_AUTO_UPGRADE=1
    volumes:
      - ./data/mariadb:/var/lib/mysql
    restart: unless-stopped
    networks:
      - seafile-net

  memcached:
    image: memcached
    container_name: seafile-memcached
    entrypoint: memcached -m 256
    restart: unless-stopped
    networks:
      - seafile-net

  seafile:
    image: seafileltd/seafile-mc
    container_name: seafile
    ports:
      - "8080:80"
#      - "443:443"
    volumes:
      - ./data/app:/shared
#     - ./seafile-data:/shared
    environment:
      - DB_HOST=db
      - DB_ROOT_PASSWD=CHANGE_THIS_TO_AOTHER_SUPER_SECRET_PW_123
      - TIME_ZONE=Etc/UTC
#      - TZ=Europe/Oslo
      - SEAFILE_ADMIN_EMAIL=your@mail.com
      - SEAFILE_ADMIN_PASSWORD=SECRET_WEB_LOGIN_PW
      - SEAFILE_SERVER_LETSENCRYPT=false
      - SEAFILE_SERVER_HOSTNAME=10.0.1.10
#      - FORCE_HTTPS_IN_CONF=true
    depends_on:
      - db
      - memcached
    restart: unless-stopped
    networks:
      - seafile-net

networks:
  seafile-net:
````

Build the shit
````
#!/bin/bash
docker pull seafileltd/seafile-mc:latest
docker compose pull
docker compose restart

#docker compose down

#docker-compose build              #old compose command
#docker-compose build --no-cache   #old compose command

docker compose up -d               #brings everything up

sudo systemctl restart nginx
sudo systemctl status nginx
````

- Another important seafile settins file is  `/home/UserName/seafile-server/data/app/seafile/conf/seahub_settings.py` and my looked something like this
````
# -*- coding: utf-8 -*-
SECRET_KEY = "random stuff removed here"
SERVICE_URL = "https://10.0.1.10"
CSRF_TRUSTED_ORIGINS = ["https://10.0.1.10"]
STATICFILES_DIRS = []
#CSRF_TRUSTED_ORIGINS = ["https://seafile.local"]

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.mysql',
        'NAME': 'seahub_db',
        'USER': 'seafile',
        'PASSWORD': 'REMOVED',
        'HOST': 'db',
        'PORT': '3306',
        'OPTIONS': {'charset': 'utf8mb4'},
    }
}


CACHES = {
    'default': {
        'BACKEND': 'django_pylibmc.memcached.PyLibMCCache',
        'LOCATION': 'memcached:11211',
    },
    'locmem': {
        'BACKEND': 'django.core.cache.backends.locmem.LocMemCache',
    },
}
COMPRESS_CACHE_BACKEND = 'locmem'
TIME_ZONE = 'Etc/UTC'
FILE_SERVER_ROOT = "https://10.0.1.10/seafhttp"
````

### Recovery of files
I tried these without luck
1. https://github.com/awant/seafile_data_recovery - didn't work in particular
2. [https://git.deuxfleurs.fr/quentin/seafile_recovery](https://git.deuxfleurs.fr/quentin/seafile_recovery) - Strugled installing golang and downloading the shit

The solution was this repo: [https://gist.github.com/JFWenisch/50c48bd4706e6762ef531b7b3041ad1c](https://gist.github.com/JFWenisch/50c48bd4706e6762ef531b7b3041ad1c). I followed the instructions to point and the files got recovered.
I follow this 
