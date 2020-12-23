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

