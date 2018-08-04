## docker-project-drupal

The Docker Compose file
In this tutorial, we'll be describing a couple of different methods for importing and exporting a database into a container, all of which can be facilitated by a docker-compose.yml file with the following contents:

```
version: '3'
services:
  web:
    image: lullaboteducation/drupaldevwithdocker-php
    volumes:
      - ./docroot:/var/www/html:cached
    ports:
      - "80:80"
  db:
    image: lullaboteducation/drupaldevwithdocker-mysql
    volumes:
      - ./db-backups:/var/mysql/backups:delegated
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: drupaldb
      MYSQL_USER: drupal
      MYSQL_PASSWORD: verybadpassword
    ports:
      - "3306:3306"
  pma:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: root
      PHP_UPLOAD_MAX_FILESIZE: 1G
      PHP_MAX_INPUT_VARS: 1G
    ports:
     - "8001:80"
```  
The rest of the tutorial will highlight certain sections of this docker-compose.yml file with a corresponding method for database import/export.

Setting the stage
Before we can start loading a database, we need to have a set of containers to use as our local development environment. For a Drupal site, we need at least two containers: One for Apache and mod_php, and another to provide us a MySQL database:
```
version: '3'
services:
  web:
    image: lullaboteducation/drupaldevwithdocker-php
    ports:
      - "80:80"
  db:
    image: lullaboteducation/drupaldevwithdocker-mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: drupaldb
      MYSQL_USER: drupal
      MYSQL_PASSWORD: verybadpassword
 ```     
Our project directory looks fairly standard for a Drupal site, too:

```
/path/to/my_project
+-- .git/
+-- docker-compose.yml
+-- docroot/
    +-- core/
    +-- index.php
 ```   
The Compose file (docker-compose.yml) is in the repository root. For convenience, our site files are in a subdirectory of the repository: docroot. This makes it much easier to define a bind volume in Docker, as well as keeps the site files separate from project-related files like docker-compose.yml.

Method #1: Using a database client

If you have a database client already installed on your host OS, such as the MySQL CLI client, or a graphical client like Sequel Pro, you can use it to load a database dump into the container.

First, we need to expose the database port on our database container:
```
version: '3'
services:
  db:
    image: lullaboteducation/drupaldevwithdocker-mysql
    ports:
      - "3306:3306"
```
Then, we start the container set as we normally would:

```
$ cd /path/to/my_project
$ docker-compose up -d
```
If using the MySQL CLI client to talk to our Compose file above, the command would look like this:
```
$ mysql -u drupal -p -C drupaldb -h 127.0.0.1 -P 3306
```
This will put you into a mysql prompt where you can run mysql commands, such as show databases; to list the databases in this container.

In this way, it's a lot like communicating with a database installed on the host OS. Since a containerized database with an exposed port looks exactly like a database installed on the host OS, the client does not know the difference. By default, a container isn't given a unique IP address. It shares the same IP with the host OS.

Using a database client on the host OS tends to work well for small to medium websites. Once the database site exceeds a certain point, however, you will start to get timeout problems that require tuning the database network configuration.

Method #2: Using a bind volume

When loading the largest of database dumps, it's often better to rely on the database CLI client on the database server itself. We can do something very similar with Docker by using a bind volume.

First we need a directory in which to share the database dump easily from the hostOS to the container. We update our Compose file to provide a directory on the host OS in which to save database dumps. This directory can be named anything, but for simplicity, let's call it db-backups/ and create it in the root of our project directory:
```
/path/to/my_project
+-- .git/
+-- db-backups/
|   +-- .gitignore
|   +-- mydb_2017-12-12.sql
+-- docker-compose.yml
+-- docroot/
    +-- core/
    +-- index.php
    
```
Note the .gitignore in our database dump directory. The .gitignore is configured to ignore *.sql files and any commonly compressed *.sql files. This prevents us from accidentally committing database dumps to the repo.
```
Example .gitignore for db-backups/:

.tar.gz
.zip
.sql
```
Next, we need to add a volumes key to our docker-compose.yml file to mount the directory as a bind volume.
```
version: '3'
services:
  db:
    image: lullaboteducation/drupaldevwithdocker-mysql
    volumes:
      - ./db-backups:/var/mysql/backups:delegated
```
Save the Compose file and restart the containers:

```
$ cd /path/to/my_project
$ docker-compose kill
$ docker-compose up -d
```
If you don't already have a database dump, create one with a tool of your choice (mysqldump, drush sql-dump, Backup and Migrate module, or favorite GUI database tool such as Sequel Pro) and save it to your project's db-backups/ directory.

Once you have your containers up and running and a database backup saved to db-backups/, use docker-compose exec to enter into the db container and load the database. Don't forget to change the filename to the name of the backup file in your db-backups/ directory. You don't need to include the db-backups/ path because of how we set up the volumes key in the docker-compose.yml file.

Example procedure (change values as appropriate to your setup):
```
$ cd /path/to/my_project
$ docker-compose exec db /bin/bash

# cd /var/mysql/backups
# mysql -u root -p -C drupaldb < mydb_2017-12-12.sql
# exit
```

By execing into the database container, we're effectively running the mysql command as if we were on the database server. This gets around network communication problems and helps avoid errors like The MySQL server has gone away. Using a bind volume in this way is effective even for multi-gigabyte databases.

Method #3: Use another container
Sometimes you don't have all the necessary tools on your host OS. This is particularly true if you needed to replace your laptop or workstation recently. Instead of installing a bunch of additional tools on the bare metal, you could instead add containers to the set that contain the tools.

Many Drupal developers are familiar with web-based SQL interfaces like phpMyAdmin. phpMyAdmin is available as a Docker container, and with a few additional lines of YAML, you can add that to your set.

First, we add the phpMyAdmin container to the docker-compose.yml file:
```
version: '3'
services:
...
  pma:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: root
      PHP_UPLOAD_MAX_FILESIZE: 1G
      PHP_MAX_INPUT_VARS: 1G
    ports:
     - "8001:80"
     
```
The pma container takes several configuration parameters as environment variables. This includes the hostname of the database server (in our case the service name of the db container), the username, password, and additional configuration options.

phpMyAdmin can be accessed by a web browser, but we already have our web server container using port 80. To avoid a port conflict, we re-map port 80 of the pma container to an alternate port on the host OS. In the above example, port 8001.

Then, all we need to do is stop and restart the container set:
```
$ cd /path/to/my_project
$ docker-compose kill
$ docker-compose up -d

```
We can then visit phpMyAdmin by opening a web browser, and navigating to http://127.0.0.1:8001. After that, we can use the tool to import the database.

Adding another container that provides a tool to work with other containers has some advantages. It's easier for less technically experienced users who can use a provided, pre-configured graphical tool. It also removes the need to install additional tooling on each team member's laptop or workstation, making moving systems much faster.

Putting it all together
We don't need to decide on only one of these methods. In many cases, configuring your containers to use both method 1 and 2 is effective. If disk and memory space isn't a concern (the pma container is size optimized out of the box), providing all three gives developers the most options. Here again is the docker-compose.yml with options that facilitate each of the methods described in this tutorial.
```
version: '3'
services:
  web:
    image: lullaboteducation/drupaldevwithdocker-php
    volumes:
      - ./docroot:/var/www/html:cached
    ports:
      - "80:80"
  db:
    image: lullaboteducation/drupaldevwithdocker-mysql
    volumes:
      - ./db-backups:/var/mysql/backups:delegated
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: drupaldb
      MYSQL_USER: drupal
      MYSQL_PASSWORD: verybadpassword
    ports:
      - "3306:3306"
  pma:
    image: phpmyadmin/phpmyadmin
    environment:
      PMA_HOST: db
      PMA_USER: root
      PMA_PASSWORD: root
      PHP_UPLOAD_MAX_FILESIZE: 1G
      PHP_MAX_INPUT_VARS: 1G
    ports:
     - "8001:80"
```
