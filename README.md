# Lab12
# Screen - phpmyadmin port 90 , phpinfo() port 80

# Files
# Apache Dockerfile
```Docker
FROM httpd:2.4.33-alpine

RUN apk update; \
    apk upgrade;

# Copy apache vhost file to proxy php requests to php-fpm container
COPY demo.apache.conf /usr/local/apache2/conf/demo.apache.conf
RUN echo "Include /usr/local/apache2/conf/demo.apache.conf" \
    >> /usr/local/apache2/conf/httpd.conf
```
# Apache config
```Docker
ServerName localhost

LoadModule deflate_module /usr/local/apache2/modules/mod_deflate.so
LoadModule proxy_module /usr/local/apache2/modules/mod_proxy.so
LoadModule proxy_fcgi_module /usr/local/apache2/modules/mod_proxy_fcgi.so

<VirtualHost *:80>
    # Proxy .php requests to port 9000 of the php-fpm container
    ProxyPassMatch ^/(.*\.php(/.*)?)$ fcgi://php:9000/var/www/html/$1
    DocumentRoot /var/www/html/
    <Directory /var/www/html/>
        DirectoryIndex index.php
        Options Indexes FollowSymLinks
        AllowOverride All
        Require all granted
    </Directory>
    
    # Send apache logs to stdout and stderr
    CustomLog /proc/self/fd/1 common
    ErrorLog /proc/self/fd/2
</VirtualHost>
```
# Php Dockerfile
```Dockerfile
FROM php:7.2.7-fpm-alpine3.7

RUN apk update; \
    apk upgrade;

RUN docker-php-ext-install mysqli
```
# index.php
```php
<? phpinfo(); ?>
```
# .env config
```Docker
PHP_VERSION=5.6
MYSQL_VERSION=5.7
APACHE_VERSION=2.4.32

DB_ROOT_PASSWORD=rootpassword
DB_NAME=dbtest
DB_USERNAME=otherUser
DB_PASSWORD=password

PROJECT_ROOT=./public_html
```
# docker-compose
```Docker
version: "3.2"
services:

  php:
    build: 
      context: './php/'
      args:
       PHP_VERSION: ${PHP_VERSION}
    networks:
      - backend
    volumes:
      - ${PROJECT_ROOT}/:/var/www/html/
    container_name: php

  phpmy:
    image: phpmyadmin
    networks:
      - backend
    ports:
      - '90:80'
    restart: 'unless-stopped'

  apache:
    build:
      context: './apache/'
      args:
       APACHE_VERSION: ${APACHE_VERSION}
    depends_on:
      - php
      - db
    networks:
      - frontend
      - backend
    ports:
      - "80:80"
    volumes:
      - ${PROJECT_ROOT}/:/var/www/html/
    container_name: apache

  db:
    image: mysql:${MYSQL_VERSION:-latest}
    restart: always
    volumes:
            - data:/var/lib/mysql
    networks:
      - backend
    environment:
      MYSQL_ROOT_PASSWORD: "${DB_ROOT_PASSWORD}"
      MYSQL_DATABASE: "${DB_NAME}"
      MYSQL_USER: "${DB_USERNAME}"
      MYSQL_PASSWORD: "${DB_PASSWORD}"
    container_name: mysql
networks:
  frontend:
  backend:
volumes:
    data:
```
