---
tags: postgresql, debian
title: Installing and configuring Postgresql 9.4 on Debian 8
---

Install the postgresql server package:

    #apt-get install postgresql-9.4

Create cluster (if it haven't created automatically):

    #pg_createcluster 9.4 main --start

Next we need to allow connects from remote sources.

Edit ```/etc/postgresql/9.4/main/postgresql.conf``` and set ```listen_addresses``` to ```'*'```.

Restart the Postgresql service:

    #service postgresql Restart


Now we should create database user.

    #su postgres
    #psql

    postgres=# CREATE USER db_user_name WITH PASSWORD 'secret_password';

Create new database and grant access to it to our user.

    postgres=# CREATE DATABASE "db_name"
      WITH OWNER "db_user_name"
      ENCODING 'UTF8'
      LC_COLLATE = 'en_US.UTF-8'
      LC_CTYPE = 'en_US.UTF-8'
      TEMPLATE = template0;

Now we have database with UTF8 collation, it is time to allow connect from remote sources. Our user is the owner of the schema.

Edit ```/etc/postgresql/9.4/main/pg_hba.conf``` and add line

    host    all   db_user_name    0.0.0.0/0               md5

after the line:

    host    all   all             127.0.0.1/32            md5


Save file and restart the service.

Now we can connect to our server from remote computer.
