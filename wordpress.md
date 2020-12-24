# Dockerizzare un wordpress esistente

[Repo di esempio](https://github.com/MaoX17/wordpress-docker-compose)

# wordpress-docker-compose
Eseguire wordpress su docker o migrare un Wordpress esistente su Docker

Se alcuni link non dovessero funzionare prova qui
https://github.com/MaoX17/wordpress-docker-compose/
o
[qui](https://github.com/MaoX17/wordpress-docker-compose/)


![GreMaPro](https://www.maurizio.proietti.name/wp-content/uploads/2020/12/cropped-gremapro_small.gif?s=150)

## Dockerizzare un wordpress esistente

Questa versione è valida sia con traefik che con nginx di jwilder come reverse proxy

Adesso iniziamo

# Quick Install ... la versione rapida


```
mkdir -p /opt/docker

git clone https://github.com/MaoX17/wordpress-docker-compose.git

cd wordpress-docker-compose

docker network create proxy

cd 00_traefik

Genero la stringa di auth:

htpasswd -n username

ottengo:

username:$apr1$VDSty0Wy$5nrZ7nthjusltZXM0eE2s/

Copio la stringa e la inserisco nel file [traefik_dynamic.toml](00_traefik/conf/traefik_dynamic.toml)

Sempre nello stesso file sostituisi

rule = "Host(`traefik.proietti.net`)"

con il tuo indirizzo per il manager di traefik

Nel file [traefik.toml](00_traefik/conf/traefik.toml)
sostituire 
email = "maurizio.proietti(AT)EMAIL.com"
Con la vostra email

Poi

docker-compose build
docker-compose up -d


```

E Traefik è installato e funzionante

Passiamo a WP

```

cp wp05/env.example wp05/.env

Modifica il file .env con i dati che vuoi

cd wp05

mkdir data/redis

cp conf/redis.conf wp05/redis/

Modifica il Dockerfile

poi

docker-compose build
docker-compose up -d


```

# Spiegone

Di seguito ci sono i passaggi per ralizzare il tutto da zero...



Partiamo dal traefik


## Traefik

Primo comando:

```
docker network create proxy

```

Genero la stringa di auth:

    htpasswd -n username

ottengo:

    username:$apr1$VDSty0Wy$5nrZ7nthjusltZXM0eE2s/

Creo il file acme.json

    touch conf/acme.json
    chmod 600 conf/acme.json


Creo il file ./conf/traefik.toml

```

[entryPoints]
  [entryPoints.web]
    address = ":80"
    [entryPoints.web.http.redirections.entryPoint]
      to = "websecure"
      scheme = "https"

  [entryPoints.websecure]
    address = ":443"

[api]
  dashboard = true

[certificatesResolvers.lets-encrypt.acme]
  email = "maurizio.proietti(AT)EMAIL.com"
  storage = "acme.json"
  [certificatesResolvers.lets-encrypt.acme.tlsChallenge]

[providers.docker]
  watch = true
  network = "proxy"


[providers.file]
  filename = "traefik_dynamic.toml"
  

```

Creo il file ./conf/traefik_dynamic.toml

```
[http.middlewares.simpleAuth.basicAuth]
  users = [
    "username:$apr1$VDSty0Wy$5nrZ7nthjusltZXM0XXeE2s/"
  ]

[http.routers.api]
  rule = "Host(`traefik.proietti.net`)"
  entrypoints = ["websecure"]
  middlewares = ["simpleAuth"]
  service = "api@internal"
  [http.routers.api.tls]
    certResolver = "lets-encrypt"


```

Creo il ./docker-compose.yaml:

```
version: "3.3"

services:

  traefik:
    image: "traefik:v2.2"
    container_name: "traefik"

    ports:
      - "80:80"
      - "443:443"

    networks:
      - "proxy"
    volumes:
      - "./data/letsencrypt:/letsencrypt"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "./conf/traefik.toml:/traefik.toml"
      - "./conf/traefik_dynamic.toml:/traefik_dynamic.toml"
      - "./conf/acme.json:/acme.json"
      


networks:
  proxy:
    external: true


```

Lancio il docker-compose up -d

```
docker-compose up -d

```

Et voilà!!!

Per vedere il risultato mi collego a 

traefik.proietti.net

e faccio login


## WordPress


Per usare il docker-compose occorre creare prima di tutto un file .env esempio [env.example](wp05/env.example)

Di seguito un esempio:

```
## WP ENV
WORDPRESS_DB_HOST=db
WORDPRESS_DB_USER=userdb
WORDPRESS_DB_PASSWORD=passworddb
WORDPRESS_DB_NAME=database
VIRTUAL_HOST=www.maurizio.proietti.name,blog.proietti.net
VIRTUAL_PORT=80
LETSENCRYPT_HOST=www.maurizio.proietti.name
LETSENCRYPT_EMAIL=maurizio.proietti(AT))EMAIL.com

## MYSQL ENV
MYSQL_DATABASE=database
MYSQL_USER=userdb
MYSQL_PASSWORD=passworddb
MYSQL_ROOT_PASSWORD=segretissima

##TRAEFIK
TRAEFIK_ROUTE_NAME=wp_mp


```

Nella dir dockerfile/wp trovo:

* il Dockerfile di WP
* la conf di apache (mpm) con alto o scarso traffico
* le impostazioni personalizzate del php.ini


Ho personalizzato un po' l'immagine di WordPress per avere un po' di performance in più e per distinguere un sito con meggior traffico da uno con meno traffico.
Per cambiare impostazione vedi (dockerfile/wp/Dockerfile) e in particolare le righe:

```
## APACHE
#COPY ./mpm_prefork_low_trafic.conf /etc/apache2/mods-available/mpm_prefork.conf
COPY ./mpm_prefork_low_trafic.conf /etc/apache2/mods-available/mpm_prefork.conf
```

Anche il php.ini è modificato per gestire le dimensioni di upload
v. dockerfile/wp/php-wp.ini



Creo il mio docker-compose.yml

```
version: '3.1'

services:


  wordpress:
#    image: wordpress
    build:
      # call the Dockerfile in ./wordpress
      context: ./dockerfile/wp
    restart: always
    container_name: wp_www.maurizio.proietti.name
    environment:
      WORDPRESS_DB_HOST: ${WORDPRESS_DB_HOST}
      WORDPRESS_DB_USER: ${WORDPRESS_DB_USER}
      WORDPRESS_DB_PASSWORD: ${WORDPRESS_DB_PASSWORD}
      WORDPRESS_DB_NAME: ${WORDPRESS_DB_NAME}
      VIRTUAL_HOST: ${VIRTUAL_HOST}
      VIRTUAL_PORT: ${VIRTUAL_PORT}
      LETSENCRYPT_HOST: ${LETSENCRYPT_HOST}
      LETSENCRYPT_EMAIL: ${LETSENCRYPT_EMAIL}
    labels:
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.rule=Host(`${LETSENCRYPT_HOST}`)
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.tls=true
      - traefik.http.routers.${TRAEFIK_ROUTE_NAME}.tls.certresolver=lets-encrypt
      - traefik.port=${VIRTUAL_PORT}
    depends_on:
      - db
      - redis
    restart: unless-stopped
    networks:
      - proxy
      - backend
    volumes:
      - ./data/html:/var/www/html

  db:
    container_name: mysql_www.maurizio.proietti.name
    image: mysql:5.7
    restart: always
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE}
      MYSQL_USER: ${MYSQL_USER}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
    labels:
      - traefik.enable=false
    volumes:
      - ./data/mysql:/var/lib/mysql
    networks:
      - backend


  redis:
    image: redis:6
    container_name: redis_www.maurizio.proietti.name
    restart: always
    sysctls:
      - net.core.somaxconn=1024
    labels:
      - traefik.enable=false
    volumes:
      - ./data/redis:/data
    networks:
      - backend
    # launch Redis in cache mode with :
    #     #  - max memory up to 50% of your RAM if needed (--maxmemory 512mb)
    #         #  - deleting oldest data when max memory is reached (--maxmemory-policy allkeys-lru)
#    entrypoint: redis-server --maxmemory 512mb -maxmemory-policy allkeys-lru
    entrypoint: redis-server /data/redis.conf




networks:
  proxy:
    external: true
  backend:
    external: false






```

Lancio:

```
docker-compose build
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


Posso poi installare w3 total cache.

Attenzione!!!

Occorre settare correttamente l'indirizzo di redis: 

```
redis:6379
```
