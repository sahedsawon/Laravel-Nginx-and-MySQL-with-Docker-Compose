# Write by   [Faizan Bashir](https://www.digitalocean.com/community/users/faizanbashir)
##### Link of the Tutorial [How To Set Up Laravel, Nginx, and MySQL with Docker Compose](https://www.digitalocean.com/community/tutorials/how-to-set-up-laravel-nginx-and-mysql-with-docker-compose)

#### Step 1 — Downloading Laravel and Installing Dependencies
```
cd ~
git clone https://github.com/laravel/laravel.git laravel-app
```
- Move into the laravel-app directory:
 ```
cd ~/laravel-app
```

- Next, use Docker’s composer image to mount the directories that you will need for your Laravel project and avoid the overhead of installing Composer globally:

```
docker run --rm -v $(pwd):/app composer install
```

- As a final step, set permissions on the project directory so that it is owned by your non-root user:
```
sudo chown -R $USER:$USER ~/laravel-app
```

### Step 2 — Creating the Docker Compose File

- Open the docker-compose.yml (if no docker file then create one )file:
```
vim ~/laravel-app/docker-compose.yml 
```
- define the Services 

In the docker-compose file, you will define three services: app, webserver, and db. Add the following code to the file, being sure to replace the root password for MYSQL_ROOT_PASSWORD, defined as an environment variable under the db service, with a strong password of your choice:
```
                   ~/laravel-app/docker-compose.yml

version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: digitalocean.com/php
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: your_mysql_root_password
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
 ```

### Step 3 — Persisting Data

- In the docker-compose file, define a volume called dbdata under the db service definition to persist the MySQL database:

```
                        ~/laravel-app/docker-compose.yml
...
#MySQL Service
db:
  ...
    volumes:
      - dbdata:/var/lib/mysql
    networks:
      - app-network
  ...
```

- At the bottom of the file, add the definition for the dbdata volume:

```
                ~/laravel-app/docker-compose.yml
...
#Volumes
volumes:
  dbdata:
    driver: local
```

- Next, add a bind mount to the db service for the MySQL configuration files you will create in Step 7:

```
                    ~/laravel-app/docker-compose.yml
...
#MySQL Service
db:
  ...
    volumes:
      - dbdata:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/my.cnf
  ...
```

- Next, add bind mounts to the webserver service.

```
                        ~/laravel-app/docker-compose.yml
#Nginx Service
webserver:
  ...
  volumes:
      - ./:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
  networks:
      - app-network
```

- Finally, add the following bind mounts to the app service for the application code and configuration files:

```
                    ~/laravel-app/docker-compose.yml
#PHP Service
app:
  ...
  volumes:
       - ./:/var/www
       - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
  networks:
      - app-network
```
##### Final docker-compose.yml file now look like this: 

```
version: '3'
services:

  #PHP Service
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: digitalocean.com/php
    container_name: app
    restart: unless-stopped
    tty: true
    environment:
      SERVICE_NAME: app
      SERVICE_TAGS: dev
    working_dir: /var/www
    volumes:
      - ./:/var/www
      - ./php/local.ini:/usr/local/etc/php/conf.d/local.ini
    networks:
      - app-network

  #Nginx Service
  webserver:
    image: nginx:alpine
    container_name: webserver
    restart: unless-stopped
    tty: true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./:/var/www
      - ./nginx/conf.d/:/etc/nginx/conf.d/
    networks:
      - app-network

  #MySQL Service
  db:
    image: mysql:5.7.22
    container_name: db
    restart: unless-stopped
    tty: true
    ports:
      - "3306:3306"
    environment:
      MYSQL_DATABASE: laravel
      MYSQL_ROOT_PASSWORD: your_mysql_root_password
      SERVICE_TAGS: dev
      SERVICE_NAME: mysql
    volumes:
      - dbdata:/var/lib/mysql/
      - ./mysql/my.cnf:/etc/mysql/my.cnf
    networks:
      - app-network

#Docker Networks
networks:
  app-network:
    driver: bridge
#Volumes
volumes:
  dbdata:
    driver: local
``` 

### Step 4 — Creating the Dockerfile

- Create a Dockerfile in the laravel-app directory and place the flowing code 

```
FROM php:7.2-fpm

# Copy composer.lock and composer.json
COPY composer.lock composer.json /var/www/

# Set working directory
WORKDIR /var/www

# Install dependencies
RUN apt-get update && apt-get install -y \
    build-essential \
    mysql-client \
    libpng-dev \
    libjpeg62-turbo-dev \
    libfreetype6-dev \
    locales \
    zip \
    jpegoptim optipng pngquant gifsicle \
    vim \
    unzip \
    git \
    curl

# Clear cache
RUN apt-get clean && rm -rf /var/lib/apt/lists/*

# Install extensions
RUN docker-php-ext-install pdo_mysql mbstring zip exif pcntl
RUN docker-php-ext-configure gd --with-gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/ --with-png-dir=/usr/include/
RUN docker-php-ext-install gd

# Install composer
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

# Add user for laravel application
RUN groupadd -g 1000 www
RUN useradd -u 1000 -ms /bin/bash -g www www

# Copy existing application directory contents
COPY . /var/www

# Copy existing application directory permissions
COPY --chown=www:www . /var/www

# Change current user to www
USER www

# Expose port 9000 and start php-fpm server
EXPOSE 9000
CMD ["php-fpm"]
```

### Step 5 — Configuring PHP
- Create the php directory:

```  
mkdir ~/laravel-app/php
```
- Next, open the local.ini file:

``` 
vim ~/laravel-app/php/local.ini
```
- add the following code to set size limitations for uploaded files:

``` 
upload_max_filesize=40M
post_max_size=40M
```

### Step 6 — Configuring Ngin

- First, create the nginx/conf.d/ directory:

``` 
mkdir -p ~/laravel-app/nginx/conf.d
```

- Create the app.conf configuration file:

``` 
vim ~/laravel-app/nginx/conf.d/app.conf
```
. Add the following code to the file to specify your Nginx configuration:

``` 
server {
    listen 80;
    index index.php index.html;
    error_log  /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
    location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
    location / {
        try_files $uri $uri/ /index.php?$query_string;
        gzip_static on;
    }
}
```

### Step 7 — Configuring MySQL

- First, create the mysql directory:

``` 
mkdir ~/laravel-app/mysql
```
- Next, make the my.cnf file:
``` 
vim ~/laravel-app/mysql/my.cnf
```
- In the file, add the following code to enable the query log and set the log file location:

``` 
[mysqld]
general_log = 1
general_log_file = /var/lib/mysql/general.log
```

### Step 8 — Running the Containers and Modifying Environment Settings

Now that you have defined all of your services in your docker-compose file and created the configuration files for these services, you can start the containers. As a final step, though, we will make a copy of the .env.example file that Laravel includes by default and name the copy .env, which is the file Laravel expects to define its environment:

``` 
cp .env.example .env
```

With all of your services defined in your docker-compose file, you just need to issue a single command to start all of the containers, create the volumes, and set up and connect the networks:


``` 
docker-compose up -d
```

Once the process is complete, use the following command to list all of the running containers:

``` 
docker ps
```

Open the file using docker-compose exec, which allows you to run specific commands in containers. In this case, you are opening the file for editing:

``` 
docker-compose exec app vim .env
```

Find the block that specifies DB_CONNECTION and update it to reflect the specifics of your setup. You will modify the following fields:

- DB_HOST will be your db database container.
- DB_DATABASE will be the laravel database.
- DB_USERNAME will be the username you will use for your database. In this case, we will use laraveluser.
- DB_PASSWORD will be the secure password you would like to use for this user account

``` 
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=laraveluser
DB_PASSWORD=your_laravel_db_password
```
- Set the application key for the Laravel application and cache these settings

``` 
docker-compose exec app php artisan key:generate
docker-compose exec app php artisan config:cache
```

### Step 9 — Creating a User for MySQL

- Access to mysql service 

``` 
docker-compose exec db bash
```

- Login to mysql 

``` 
mysql -u root -p
```
-  Run the show databases command to check for existing databases

``` 
show databases;
```

- Next, create the user account that will be allowed to access this database. 

``` 
GRANT ALL ON laravel.* TO 'laraveluser'@'%' IDENTIFIED BY 'your_laravel_db_password';
```

- Flush the privileges to notify the MySQL server of the changes:

``` 
FLUSH PRIVILEGES;
exit
exit
```

### Step 10 — Migrating Data and Working with the Tinker Console

- Rnn the migration 

``` 
docker-compose exec app php artisan migrate
```

Once the migration is complete, you can run a query to check if you are properly connected to the database using the tinker command:

``` 
docker-compose exec app php artisan tinker
\DB::table('migrations')->get();
```
