#Backup:

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