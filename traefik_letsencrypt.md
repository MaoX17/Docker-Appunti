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
e qui incollo la stringa di auth che ho creato prima e cambio l'indirizzo della gui di treafik

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