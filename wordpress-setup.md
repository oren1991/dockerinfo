# Wordpress local setup with docker-compose

## run this code in your folder you have yml file
```console
docker-compose -f wordpress-compose.yml up #This will start the database and wordpress
```


## Log in to wordpress container
default code: 
```console
docker exec -it wordpress_wordpress_1 bash
```

## Give access and allow filechanges within the container
```console 
chown -R www-data:www-data /var/www/html
chmod -R 777 /var/www/html
```

## To install plugin you have to disable FTP upload go to your folder find wp-config.php add this line
```php
define('FS_METHOD','direct');
```
