# Docker info

## Install docker

### Creating docker network based on nginx-proxy
This way containers can be reached via virtual hostnames and nginx will handle which container send the request to based on the virtual host setup

```console
foo@bar:~$ docker network create nginx-proxy #this is the name of the networdk docker would use
```
Installing nginx-proxy with docker

```console
foo@bar:~$ docker run -d -p 80:80 --name nginx-proxy --net nginx-proxy -v /var/run/docker.sock:/tmp/docker.sock jwilder/nginx-proxy
```

port: 80 exposed in docker and listening on 80 of the server

Example app creation on the network

```console
foo@bar:~$ docker run -d --name blog --expose 80 --net nginx-proxy -e VIRTUAL_HOST=blog.DOMAIN.TLD wordpress
```
OR 

Compose:

```yml
version: "3"

services:
   db_node_domain:
     image: mysql:5.7
     volumes:
       - db_data:/var/lib/mysql
     restart: always
     environment:
       MYSQL_ROOT_PASSWORD: PASSWORD
       MYSQL_DATABASE: wordpress
       MYSQL_USER: wordpress
       MYSQL_PASSWORD: PASSWORD
     container_name: wordpress_db

   wordpress:
     depends_on:
       - db_node_domain
     image: wordpress:latest
     expose:
       - 80
     restart: always
     environment:
       VIRTUAL_HOST: blog.DOMAIN.TLD
       WORDPRESS_DB_HOST: db_node_domain:3306
       WORDPRESS_DB_USER: wordpress
       WORDPRESS_DB_PASSWORD: PASSWORD
     container_name: wordpress
volumes:
    db_data:

networks:
  default:
    external:
      name: nginx-proxy
```
