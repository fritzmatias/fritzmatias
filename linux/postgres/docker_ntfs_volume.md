# How to manage Postgres image with local ntfs persistence

## Introduction

This guide will walk you through the process of setting up a Dockerized PostgreSQL instance with persistent storage on an NTFS file system. This configuration ensures that your database data persists even after the container is stopped or removed.

## Prerequisites

- Docker: Ensure Docker is installed and running on your system.
- NTFS File System: A local NTFS file system where you want to store the PostgreSQL data.
- Read the postgres image [README](https://github.com/docker-library/docs/tree/master/postgres/README.md) to be on the same page before reading the guide.

## Step by step Guide

1. Create a directory on your NTFS file system to store the PostgreSQL data.

   ```sh
   # is going to be mounted by the container to persist data
   $ mkdir -p /media/ntfsRoot/postgresql/data
   ```

2. On mounted root NTFS filesystems check you have the rigth permitions for the partition.

   ```sh
   $ ls -ld /media/ntfsRoot/
   drwxrwxrwx 1 ntfsUser users 4096 Sep 26 22:02
   $ id ntfsUser
   uid=1000(ntfsRoot) gid=1000(ntfsRoot) groups=1000(ntfsroot)
   ```

   a. if you get full access permission on the ntfs root file system, [postgres initdb](initdb) will fail to start because postgres requires `data` folder and files to have rigths 750, owned by postgres user and group. So it is required to change that at mounting time. Add to `/etc/fstab` file, [ntfs options](https://manpages.debian.org/bookworm/mount/mount.8.en.html#Mount_options_for_ntfs) `rw,permissions, fmask=037,dmask=037`.

   ```text
   # /etc/fstab
   UUID=YourUUID /media/ntfsRoot     ntfs  rw,auto,user,permissions,fmask=027,dmask=027,exec,uid=1000,gid=100,errors=remount-ro 0
   ```

   After remount, or reboot your system. You should see the postgres expected output.

   ```sh
   $ ls -ld /media/ntfsRoot/
   drwxr----- 1 ntfsUser users 4096 Sep 26 22:04
   ```

3. To bind local `postgres/data` directory (in the ntfsRoot partition) to `/var/lib/postgresql/data` and have postgres running, it is required to map the directory uid & gid to postgres user and postgres group. Theorically are many approaches to do it as [documentations describes](https://github.com/docker-library/docs/blob/master/postgres/README.md#arbitrary---user-notes), like binding alternative `/etc/passwd` and `/etc/group` files, or using [cwrap](https://cwrap.org/nss_wrapper.html).
But the direct way i found to do that is using `sed`, to modify `/etc/passwd` and `/etc/group` as `root` and run the container entrypoint cmd.

   In my case, my fstab uid=1000 and my gid=100, so i need to do that switch inside the container before to launch postgres itself.

   ```sh
   # to get the current uid & gid from the container
   $ docker run postgres /bin/bash -c "grep postgres /etc/passwd /etc/group"
   /etc/passwd:postgres:x:999:999::/var/lib/postgresql:/bin/bash
   /etc/group:postgres:x:999:
   /etc/group:ssl-cert:x:101:postgres
   ```

   So, i need to map inside the container uid=999 to uid=1000 & gid=999 to gid=100, because the host inodes are going to have those values at mount point.

   ```sh
   $ docker run  -v /media/ntfsRoot/postgres/data:/var/lib/postgresql/data:/var/lib/postgresql/data postgres /bin/bash -c "sed -e 's/999/1000/g' /etc/passwd|grep postgres"
   postgres:x:1000:1000::/var/lib/postgresql:/bin/bash

   $ docker run  -v /media/ntfsRoot/postgres/data:/var/lib/postgresql/data:/var/lib/postgresql/data postgres /bin/bash -c "sed -e 's/999/100/g' /etc/group|grep postgres"
   postgres:x:100:
   ```

   To apply the sed changes in place is required to use `-i` option.

4. Replicate the image entrypoint & cmd to launch postgres after image on the fly changes.

   As the entrypoint & cmd are complementary, we can re use those values from [Postgres Dockerfile](https://github.com/docker-library/postgres/blob/0b87a9bbd23f56b1e9e863ecda5cc9e66416c4e0/17/bookworm/Dockerfile) and contatenate them as one call `docker-entrypoint.sh postgres`.

   ```sh
   $ docker run  -v /media/ntfsRoot/postgres/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres /bin/bash -c "sed -i -e 's/999/1000/g' /etc/passwd;sed -i -e 's/999/100/g' /etc/group;docker-entrypoint.sh postgres"
   ```

5. Tests

   - Using `psql` from inside container image.

   ```sh
   $ docker exec -it $(docker ps |grep postgres|cut -d' ' -f1) psql -h localhost -U postgres
   psql (17.2 (Debian 17.2-1.pgdg120+1))
   Type "help" for help.

   postgres=# 

   ```

   - Checking process ownership

   ```sh
   $  ps faxun | grep postgres
   1000  449440  0.0  0.0 2068616 31036 pts/5   Sl   14:40   0:00  |       \_ docker run -v /media/ntfsRoot/postgres/data:/var/lib/postgresql/data -e POSTGRES_PASSWORD=password postgres /bin/bash -c sed -i -e 's/999/1000/g' /etc/passwd;sed -i -e 's/999/100/g' /etc/group;docker-entrypoint.sh postgres
   1000  455675  0.0  0.0   6456  2388 pts/5    S+   14:45   0:00  |       \_ grep postgres
   1000  449576  0.0  0.0 221056 29876 ?        Ss   14:40   0:00  \_ postgres
   1000  449695  0.0  0.0 221192  5752 ?        Ss   14:40   0:00      \_ postgres: checkpointer 
   1000  449696  0.0  0.0 221208  6872 ?        Ss   14:40   0:00      \_ postgres: background writer 
   1000  449698  0.0  0.0 221056  9940 ?        Ss   14:40   0:00      \_ postgres: walwriter 
   1000  449699  0.0  0.0 222628  8612 ?        Ss   14:40   0:00      \_ postgres: autovacuum launcher 
   1000  449700  0.0  0.0 222636  8084 ?        Ss   14:40   0:00      \_ postgres: logical replication launcher 

   ```

6. (Optional) Set docker-compose.yaml

   ```yaml
   #docker-compose.yaml
   services:
      postgres:
         image: postgres
         entrypoint: ["/bin/bash"]
         command: [
            "-c", 
            "sed -i -e 's/999/1000/g' /etc/passwd;
            sed -i -e 's/999/100/g' /etc/group;
            docker-entrypoint.sh postgres"
            ]
         environment:
            - POSTGRES_PASSWORD=password
         volumes:
            - /media/ntfsRoot/postgres/data:/var/lib/postgresql/data
   ```

   ```sh
   docker compose -f docker-compose.yaml up postgres
   ```

## references 
- https://github.com/docker-library/docs/blob/master/postgres/README.md
- https://cwrap.org/nss_wrapper.html
- https://github.com/docker-library/postgres/blob/0b87a9bbd23f56b1e9e863ecda5cc9e66416c4e0/17/bookworm/Dockerfile
- https://wiki.debian.org/NTFS
- https://manpages.debian.org/bookworm/mount/mount.8.en.html#Mount_options_for_ntfs