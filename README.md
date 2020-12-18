# Docker-Appunti

Backup:

```
  docker run --rm -v /path/to/dokuwiki-backups:/backups --volumes-from dokuwiki busybox \
    cp -a /bitnami/dokuwiki /backups/latest
```

Run the backup command
We need to mount two volumes in a container we will use to create the backup: a directory on your host to store the backup in, and the volumes from the container we just stopped so we can access the data.

```
  $ docker run --rm -v /path/to/dokuwiki-backups:/backups --volumes-from dokuwiki busybox \
    cp -a /bitnami/dokuwiki /backups/latest
```

Restoring a backup is as simple as mounting the backup as volumes in the containers.

```
    $ docker run -d --name  \
    ...
  -  --volume /path/to/-persistence:/bitnami/dokuwiki \
  +  --volume /path/to/-backups/latest:/bitnami/dokuwiki \
    bitnami/:latest
```

# Dockerizzare un wordpress esistente

[Link all'articolo completo](https://www.maurizio.proietti.name/2020/12/17/come-ti-dockerizzo-un-wordpress-esistente/)


Ho avuto qualche difficoltà ma poi ho trovato la via giusta.

Segui ESATTAMENTE i passaggi che riporto.

1.) Creo il mio docker-compose.yml

```
  version: '3.1'
  services:
    wordpress:
      image: wordpress
      restart: always
      container_name: wp_www.maurizio.proietti.name
      environment:
        WORDPRESS_DB_HOST: db
        WORDPRESS_DB_USER: user
        WORDPRESS_DB_PASSWORD: password123
        WORDPRESS_DB_NAME: db
        VIRTUAL_HOST: www.maurizio.proietti.name
        VIRTUAL_PORT: 80
        LETSENCRYPT_HOST: www.maurizio.proietti.name
        LETSENCRYPT_EMAIL: maurizio.proietti@gmail.com
      depends_on:
        - db
      restart: unless-stopped
      networks:
        - proxy
        - www.maurizio.proietti.name-net
      volumes:
        - ./data/html:/var/www/html
    db:
      container_name: mysql_www.maurizio.proietti.name
      image: mysql:5.7
      restart: always
      environment:
        MYSQL_DATABASE: db
        MYSQL_USER: user
        MYSQL_PASSWORD: password123
        MYSQL_ROOT_PASSWORD: secret123
      volumes:
        - ./data/mysql:/var/lib/mysql
      networks:
        - www.maurizio.proietti.name-net
  networks:
    proxy:
      external:
        name: nginx-proxy
    www.maurizio.proietti.name-net:

```

Lancio il docker-compose up -d

```
docker-compose up -d
```

Entro nel sito e completo l’installazione con dati casuali

Poi faccio un rsync della SOLA directory wp-content:

```
rsync -uazv /var/www/maurizioproietti/wp/wp-content data/html/
``` 

Poi eseguo il dump del vecchio DB:

```
mysqldump --opt maurizioproietti > dump.sql
```

E controllo che la prefix delle tabelle sia wp_

Se non lo fosse sostituisco la prefix che ha il dump con wp_

Poi importo il dump nel nuovo db sotto docker:
```
cat dump.sql | docker exec -i mysql_www.maurizio.proietti.name /usr/bin/mysql -u root --password=secret123 db
```
Imposto i permessi sul filesystem per bene oppure (se ho fretta)
```
chmod -R 777 data
```
Entro nella sezione wp-admin e inizio gli aggiornamenti suggeriti nel seguente ordine (che penso possa variare la per scaramanzia non vario 🙂 )

1.) Plugins

2.) Temi

3.) WordPress Core




# Dockerizzare laravel esistente
[Link all'articolo completo](https://www.maurizio.proietti.name/2020/12/17/come-ti-dockerizzo-un-progetto-laravel-esistente/)

La mia base di partenza è stata il seguente git (che ringrazio per l’idea)

https://github.com/aschmelyun/docker-compose-laravel

Quindi ho seguito i seguenti passaggi:

```
docker network create nginx-proxy


mkdir /opt/docker
mkdir 00_nginx_reverse

cat 00_nginx_reverse/docker-compose.yml

version: '2'

services:
  nginx-proxy:
    image: jwilder/nginx-proxy
    container_name: nginx-proxy
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./data/conf:/etc/nginx/conf.d
      - ./data/vhost:/etc/nginx/vhost.d
      - ./data/dhparam:/etc/nginx/dhparam
      - ./data/nginx-html:/usr/share/nginx/html
      - ./data/certs:/etc/nginx/certs:ro
      - /var/run/docker.sock:/tmp/docker.sock:ro
    networks:
      - proxy
    restart: always

  letsencrypt:
    image: jrcs/letsencrypt-nginx-proxy-companion
    container_name: nginx-proxy-le
    environment:
      - "DEFAULT_EMAIL=maurizio.proietti@gmail.com"
      - "NGINX_PROXY_CONTAINER=nginx-proxy"
#      - "DEBUG=true"
    volumes_from:
      - nginx-proxy
    volumes:
      - ./data/vhost:/etc/nginx/vhost.d
      - ./data/nginx-html:/usr/share/nginx/html
      - ./data/certs:/etc/nginx/certs:rw
      - /var/run/docker.sock:/var/run/docker.sock:ro
    restart: always

networks:
  proxy:
    external:
      name: nginx-proxy

git clone https://github.com/aschmelyun/docker-compose-laravel.git

mv docker-compose-laravel template_laravel

cd template_laravel

vim docker-compose.yml


version: '3'

networks:
  laravel:

services:
  site:
    build:
      context: .
      dockerfile: nginx.dockerfile
    container_name: nginx_TEMPLATE_DA_CAMBIARE
    environment:
      VIRTUAL_HOST: TEMPLATE_DA_CAMBIARE
      VIRTUAL_PORT: 80
      LETSENCRYPT_HOST: TEMPLATE_DA_CAMBIARE
      LETSENCRYPT_EMAIL: maurizio.proietti@gmail.com
    volumes:
      - ./src:/var/www/html:delegated
    depends_on:
      - php
      - mysql
    networks:
      - laravel_TEMPLATE_DA_CAMBIARE
      - proxy

  mysql:
    image: mariadb
    container_name: mysql_TEMPLATE_DA_CAMBIARE
    restart: unless-stopped
    tty: true
    environment:
      MYSQL_DATABASE: homestead
      MYSQL_USER: homestead
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: secret
      SERVICE_NAME: mysql
    volumes:
      - ./mysqldata:/var/lib/mysql
    networks:
      - laravel_TEMPLATE_DA_CAMBIARE

  php:
    build:
      context: .
      dockerfile: php.dockerfile
    container_name: php_TEMPLATE_DA_CAMBIARE
    volumes:
      - ./src:/var/www/html:delegated
    networks:
      - laravel_TEMPLATE_DA_CAMBIARE

  composer:
    build:
      context: .
      dockerfile: composer.dockerfile
    container_name: composer_TEMPLATE_DA_CAMBIARE
    volumes:
      - ./src:/var/www/html
    working_dir: /var/www/html
    depends_on:
      - php
    user: laravel
    networks:
      - laravel_TEMPLATE_DA_CAMBIARE
    entrypoint: ['composer', '--ignore-platform-reqs']

  npm:
    image: node:13.7
    container_name: npm_TEMPLATE_DA_CAMBIARE
    volumes:
      - ./src:/var/www/html
    working_dir: /var/www/html
    entrypoint: ['npm']

  artisan:
    build:
      context: .
      dockerfile: php.dockerfile
    container_name: artisan_TEMPLATE_DA_CAMBIARE
    volumes:
      - ./src:/var/www/html:delegated
    depends_on:
      - mysql
    working_dir: /var/www/html
    user: laravel
    entrypoint: ['php', '/var/www/html/artisan']
    networks:
      - laravel_TEMPLATE_DA_CAMBIARE

networks:
  proxy:
    external:
      name: nginx-proxy
  laravel_TEMPLATE_DA_CAMBIARE:


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

Poi possiamo costruire i container:

```
docker-compose up --build 
```

Occorre migrare il db:

```
mysqldump --opt db_original > dump.sql

cat dump.sql | docker exec -i mysql_TEMPLATE_DA_CAMBIARE  /usr/bin/mysql -u root --password=secret homestead

```
