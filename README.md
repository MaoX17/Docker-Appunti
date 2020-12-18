# Docker-Appunti

Backup:
<code>
docker run --rm -v /path/to/dokuwiki-backups:/backups --volumes-from dokuwiki busybox \
  cp -a /bitnami/dokuwiki /backups/latest
</code> 

Step 2: Run the backup command
We need to mount two volumes in a container we will use to create the backup: a directory on your host to store the backup in, and the volumes from the container we just stopped so we can access the data.

<code>
$ docker run --rm -v /path/to/dokuwiki-backups:/backups --volumes-from dokuwiki busybox \
  cp -a /bitnami/dokuwiki /backups/latest

</code>

Restoring a backup
Restoring a backup is as simple as mounting the backup as volumes in the containers.

For the DokuWiki container:

<code>
 $ docker run -d --name  \
   ...
-  --volume /path/to/-persistence:/bitnami/dokuwiki \
+  --volume /path/to/-backups/latest:/bitnami/dokuwiki \
   bitnami/:latest
</code>
