# Dockerizzare laravel esistente


La mia base di partenza è stata il seguente git (che ringrazio per l’idea)

https://github.com/aschmelyun/docker-compose-laravel

Si da per scontato che abbiate già installato traefik o nginx

```
vim docker-compose.yml

version: '3'

services:
  site:
    build:
      context: .
      dockerfile: nginx.dockerfile
    container_name: nginx_www.strails.it
    environment:
      VIRTUAL_HOST: ${VIRTUAL_HOST}
      VIRTUAL_PORT: ${VIRTUAL_PORT}
      LETSENCRYPT_HOST: ${LETSENCRYPT_HOST}
      LETSENCRYPT_EMAIL: ${LETSENCRYPT_EMAIL}
    labels:
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.rule=Host(`${LETSENCRYPT_HOST}`)
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.tls=true
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.tls.certresolver=lets-encrypt
      - traefik.port=${VIRTUAL_PORT}
    volumes:
      - ./src:/var/www/html:delegated
    depends_on:
      - php
      - mysql
    networks:
      - backend
      - proxy

  mysql:
    image: mariadb
    container_name: mysql_www.strails.it
    restart: unless-stopped
    tty: true
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
      SERVICE_NAME: ${SERVICE_NAME}
    labels:
      - traefik.enable=false
    volumes:
      - ./mysqldata:/var/lib/mysql
    networks:
      - backend

  php:
    build:
      context: .
      dockerfile: php.dockerfile
    container_name: php_www.strails.it
    labels:
      - traefik.enable=false
    volumes:
      - ./src:/var/www/html:delegated
    networks:
      - backend

  # composer:
  #   build:
  #     context: .
  #     dockerfile: composer.dockerfile
  #   container_name: composer_www.strails.it
  #   labels:
  #     - traefik.enable=false
  #   volumes:
  #     - ./src:/var/www/html
  #   working_dir: /var/www/html
  #   depends_on:
  #     - php
  #   user: laravel
  #   networks:
  #     - backend
  #   entrypoint: ['composer', '--ignore-platform-reqs']

  # npm:
  #   image: node:13.7
  #   container_name: npm_www.strails.it
  #   labels:
  #     - traefik.enable=false
  #   volumes:
  #     - ./src:/var/www/html
  #   working_dir: /var/www/html
  #   entrypoint: ['npm']

  # artisan:
  #   build:
  #     context: .
  #     dockerfile: php.dockerfile
  #   container_name: artisan_www.strails.it
  #   labels:
  #     - traefik.enable=false
  #   volumes:
  #     - ./src:/var/www/html:delegated
  #   depends_on:
  #     - mysql
  #   working_dir: /var/www/html
  #   user: laravel
  #   entrypoint: ['php', '/var/www/html/artisan']
  #   networks:
  #     - backend

  redis:
    image: redis:6
    container_name: redis_www.strails.it
    labels:
      - traefik.enable=false
    restart: always
    sysctls:
      - net.core.somaxconn=1024
    volumes:
      - ./redis:/data
    networks:
      - backend
    entrypoint: redis-server /data/redis.conf



networks:
  proxy:
    external: true
  backend:
    external: false




mkdir redis
cd redis

vim redis.conf

maxmemory 512mb
maxmemory-policy allkeys-lru

cd ..

vim php.dockerfile

FROM php:7.2-fpm-alpine

ADD ./php/www.conf /usr/local/etc/php-fpm.d/www.conf

RUN addgroup -g 1000 laravel && adduser -G laravel -g laravel -s /bin/sh -D laravel

RUN mkdir -p /var/www/html

RUN chown laravel:laravel /var/www/html

WORKDIR /var/www/html

RUN docker-php-ext-install pdo pdo_mysql

RUN apk update

RUN apk add libpng libpng-dev libjpeg-turbo-dev libwebp-dev zlib-dev libxpm-dev gd && docker-php-ext-install gd

```

Ho dovuto installare gd perchè non c’era

Nel progetto non funzionavano i link creati con l’helper url() perchè puntavano tutti a http invece che https

Occorre quindi effettuare una modifica:

Editare il file /app/Providers/AppServiceProvider.php 

e modificare la

```
public function boot()

```
con

```
use Illuminate\Routing\UrlGenerator;


public function boot(UrlGenerator $url)
{
    if (env('APP_ENV') !== 'NATIVO') {
        $url->forceScheme('https');
    }
}
```
In questo modo i link hanno ricominciato a funzionare

Dopo occorre modificare il file .env

```
DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```

I file sorgenti di laravel vanno messi in ./src

Per dare un po' di velocità in più al nginx ho modificato il default.conf come segue:

```
server {
    listen 80;
    index index.php index.html;
    server_name localhost;
    root /var/www/html/public;




    location ~* \.(jpg|jpeg|gif|png|css|js|ico|xml)$ {
         access_log        off;
         log_not_found     off;
         expires           30d;
         #expires           max;
    }

    open_file_cache          max=2000 inactive=20s;
    open_file_cache_valid    60s;
    open_file_cache_min_uses 5;
    open_file_cache_errors   off;

    gzip  on;
    gzip_vary on;
    gzip_min_length 1024;
    #gzip_min_length 10240;
    gzip_buffers      16 8k;
    gzip_comp_level   1;
    gzip_http_version 1.1;
    gzip_proxied expired no-cache no-store private auth;

    gzip_types
    text/css
    text/javascript
    text/xml
    text/plain
    text/x-component
    application/javascript
    application/json
    application/xml
    application/rss+xml
    font/truetype
    font/opentype
    application/vnd.ms-fontobject
    image/svg+xml;

    gzip_static on;

    gzip_disable "MSIE [1-6]\.";





    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}

```


Poi possiamo costruire i container:

```
docker-compose up --build 
```

Occorre migrare il db:

```
mysqldump --opt db_original > dump.sql

cat dump.sql | docker exec -i mysql_NOME_CONTAINER  /usr/bin/mysql -u root --password=secret homestead

```


FATTOOOOO!!!!