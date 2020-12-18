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

Ho avuto qualche difficoltà ma poi ho trovato la via giusta.

Segui ESATTAMENTE i passaggi che riporto.

1.) Creo il mio docker-compose.yml


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


Lancio il docker-compose up -d

<code>
docker-compose up -d
</code>

Entro nel sito e completo l’installazione con dati casuali

Poi faccio un rsync della SOLA directory wp-content:

<code>
rsync -uazv /var/www/maurizioproietti/wp/wp-content data/html/
</code>  

Poi eseguo il dump del vecchio DB:

<code>
mysqldump --opt maurizioproietti > dump.sql
</code>  

E controllo che la prefix delle tabelle sia wp_

Se non lo fosse sostituisco la prefix che ha il dump con wp_

Poi importo il dump nel nuovo db sotto docker:
<code>
cat dump.sql | docker exec -i mysql_www.maurizio.proietti.name /usr/bin/mysql -u root --password=secret123 db
</code>  
Imposto i permessi sul filesystem per bene oppure (se ho fretta)
<code>
chmod -R 777 data
</code>  
Entro nella sezione wp-admin e inizio gli aggiornamenti suggeriti nel seguente ordine (che penso possa variare la per scaramanzia non vario 🙂 )

1.) Plugins

2.) Temi

3.) WordPress Core
