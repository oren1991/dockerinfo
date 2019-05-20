# Docker info this is modified by branch_2 and branch_1

https://docs.docker.com/install/linux/docker-ce/ubuntu/#install-docker-ce-1
## Install docker
```console
$ sudo apt-get remove docker docker-engine docker.io #remove old versions
$ sudo apt-get update
$ sudo apt-get install apt-transport-https ca-certificates curl software-properties-common #these are the dependencies
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
$ sudo apt-key fingerprint 0EBFCD88
__________________________this is info that would be printed_______________________________________________
|pub   4096R/0EBFCD88 2017-02-22
|      Key fingerprint = 9DC8 5822 9FC7 DD38 854A  E2D8 8D81 803C 0EBF CD88
|uid                  Docker Release (CE deb) <docker@docker.com>
|sub   4096R/F273FCD8 2017-02-22
I__________________________________________________________________________________________________________

$ sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
$ sudo apt-get update
$ sudo apt-get install docker-ce
$ sudo docker run hello-world #for test whether it's up
```


https://blog.ssdnodes.com/blog/host-multiple-websites-docker-nginx/
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

Compose yml file

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
```console
foo@bar:~$ docker-compose up -d #you have to install docker-composer
```

https://nodejs.org/en/docs/guides/nodejs-docker-webapp/
### To put up a node server
Have a folder with your node app
create a Dockerfile with the following

```dockerfile
FROM node:8

# Create app directory
WORKDIR /usr/src/app

# Install app dependencies
# A wildcard is used to ensure both package.json AND package-lock.json are copied
# where available (npm@5+)
COPY package*.json ./

RUN npm install
# If you are building your code for production
# RUN npm install --only=production

# Bundle app source
COPY . .

EXPOSE 8080
CMD [ "npm", "start" ]
```
.dockerignore file should be in the folder
```dockerignore
node_modules
npm-debug.log
```

## Useful docker commands
```command
$ docker images #list images
$ docker container ls #list active containers
$ docker container ls -a #to list all containers (nonactive ones also)
$ docker stop [id] #stop container
$ docker rm [id] #remove container
$ docker run [image] #shoot up a container based on an image
```


## Applying SSL

Create these folders
```console
mkdir -p /etc/nginx/certs
mkdir -p /usr/share/nginx/html
cd /etc/nginx
touch vhost.d
```

nginx-proxy has to be started with some additional volumes
```console
$ docker run -d -p 80:80 -p 443:443 --net nginx-proxy\
    --name nginx-proxy \
    -v /etc/nginx/certs:/etc/nginx/certs:ro \
    -v /etc/nginx/vhost.d \
    -v /usr/share/nginx/html \
    -v /var/run/docker.sock:/tmp/docker.sock:ro \
    --label com.github.jrcs.letsencrypt_nginx_proxy_companion.nginx_proxy \
    jwilder/nginx-proxy
```
Then start the companion container with the shared volumes

```console
$ docker run -d \
    -v /etc/nginx/certs:/etc/nginx/certs:rw \
    -v /var/run/docker.sock:/var/run/docker.sock:ro \
    --volumes-from nginx-proxy \
    jrcs/letsencrypt-nginx-proxy-companion
```

### An example app to apply to the proxy and ensure https connection
```console
$ docker run -d --name foodapp \ #container name in docker
	--expose 8080\ #PORT on which your application communicates
	--net nginx-proxy\ #apply the network where the nginx-proxy communicates
	-e VIRTUAL_HOST=food-app.progporonty.hu\ #virtual host can be applied with comma separation (no space)
	-e LETSENCRYPT_HOST=food-app.progporonty.hu\ #virtual host
	-e LETSENCRYPT_EMAIL=arbel.oren91@gmail.com\ #notification email
	vuejs/foodapp #image name
```

## Blue-Green Deployment with docker nginx-proxy
### Set up app to get latest codechanges from github on restart.

Dockerfile:
```dockerfile
FROM node

# install simple http server for serving static content
ENV NODE_VERSION stable
ENV NPM_SCRIPT ""
ENV GIT_URL https://github.com/oren1991/food-api
 
ENV APP_HOME .
ENV APP_STARTUP "server.js"
# JUST_RUN specifies whether node should be installed and git should be cloned
ENV JUST_RUN N
 
COPY ./docker-work /code
 
WORKDIR /code
 
#RUN chown -R app:app /code/*
RUN chmod +x /code/install-entrypoint.sh
RUN npm install -g n --silent
RUN n stable
 
EXPOSE 3001
ENTRYPOINT ["/code/install-entrypoint.sh"]
```
intall-entypoint.sh file
```bash
#!/bin/bash
 
if [ "$JUST_RUN" = "N" ]; then
  echo switching node to version $NODE_VERSION
  n $NODE_VERSION --quiet
fi
 
echo node version: `node --version`
 
if [ -d "app" ]; then
  # Control will enter here if $DIRECTORY exists.
  rm -rf app
fi 
 
if [ "$JUST_RUN" = "N" ]; then
  git clone $GIT_URL app
fi

mv .env ./app/.env
cd app
 
cd $APP_HOME
echo Application Home: $APP_HOME
 
if [ "$JUST_RUN" = "N" ]; then
  if [ "$YARN_INSTALL" = "1" ]; then
    yarn install --production --silent
  else
    npm install  --silent
  fi
fi
 
if [ "$APP_STARTUP" = "" ]; then
    echo "this runs npmscript" 
  npm run $NPM_SCRIPT
else
echo "this runs app_startup"
  node $APP_STARTUP
fi

```

### Deploying the app
* Deploy the app regularly to the nginx network with the virtual hosts provided above..
* Have a second container with the same settings.
* If you have to change code start that container once it's up you can stop the old
* nginx-proxy handles the routing
