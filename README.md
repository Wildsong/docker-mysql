# docker-mysql

I am sharing MySQL between a Wiki and a mailserver.

I split it out from the docker-mediawiki project.
If I had more resources I'd run separate instances for each but this is a VPS with 1GB of RAM.

I gave up on running it in swarm mode. Compose is just easier for small setups.

## Set up

Official docs: https://dev.mysql.com/doc/refman/8.0/en/linux-installation-docker.html

The data is persisted in a volume sql-data, if it does not exist it
will be created on the first start up.

Test the configuration like this.

   dc config

## Deploy

Generic...

   dc up -d

The first time you run it, it will create a random root password.  You
have to find it in the log, look for "GENERATED ROOT PASSWORD", for
example.  If you miss it, you have to remove the volume and try
again. When the server starts it will create a new volume and a new
password.

Finding the password and save it someplace, you only get one shot at this.

   dc logs mysql_database 2>&1 | grep "GENERATED ROOT PASSWORD"

Once it's running you should be able to connect to it, since it's only got a root user at this point,
you have to connect to the container itself and run the mysql command.

   docker exec -it CONTAINERID mysql -uroot -p$YOURPASS 

With that command working you should be able to load data from SQL
dumps, create users, etc.  For example, create a wikiuser account
and allow it access from anywhere on the wiki database.

   CREATE DATABASE wiki_tarra;
   CREATE USER wikitarra IDENTIFIED BY 'SECRET';
   GRANT ALL ON wiki_tarra.* TO wikitarra@'%';

Now you should be able to load data into it. See Backup and Restore section below.

### Connect from another container

Since it's running on the "database" network and it's named "mysql" there, see the compose.yaml file,
you should be able to connect to it from another container as long as it can see the database network.

   docker run --rm -it --network database debian bash
   apt-get update
   apt-get install inetutils-ping
   ping mysql
   
### Backup and restore

Let's copy the data out of the old mariadb running in mediawiki, using
mysqldump to create a file "backup/mediawiki.sql".

   docker exec `docker ps | grep mariadb | cut -b-12` mysqldump --user=wikiuser --password=THEPASSWORD wiki > backup/wiki.sql

Now we can stop mediawiki and start mysql.

   cd ../mediawiki
   docker compose down
   cd ../mysql
   docker stack deploy -c compose.yaml mysql

Now let's load the data into the new mysql instance. The '-i' tells it
to accept input from stdin. (Leave off the 't' which tells it to
accept input from a tty.)

   docker exec -i CONTAINER ID mysql --protocol tcp -uwikitarra -p$YOURPASS wiki_tarra < BACKUPS/wiki.sql

Now we can remove the maridb entry from mediawiki and restart it using the new mysql.
But that will be documented in the README over there in docker-mediawiki.

## Backups

TODO!!! SOON

## Web management

TODO -- I want a web management interface here, in spite of the overhead.
"phpmyadmin" is pretty nice.

